---
layout: post
title: "A warm proxy in front of a remote MCP server: the reentrant-client singleton"
date: 2026-07-23
categories: ai agents productivity
series: "AI coding agent productivity"
---

I stripped the Model Context Protocol (MCP) servers out of my coding agent to reclaim context window — loading twenty-odd tool schemas upfront, every session, before the model has even read the task, is a tax you pay whether or not you touch those tools. Sessions got noticeably lighter. But I'd traded one cost for another: the shell scripts I used instead now re-authenticate to a *remote* MCP server on every single invocation.

Each `read_message.py <link>` opened a fresh TLS connection to the hosted server, loaded the persisted OAuth token, ran the MCP `initialize` handshake, made one tool call, and tore the whole thing down. Death by a thousand handshakes. The obvious fix is the same one you'd reach for with any expensive-to-open connection: keep one open and reuse it.

## The shape: a local warm proxy

Put a small always-on daemon on loopback. It holds **one** persistent, authenticated upstream connection to the remote MCP server and re-serves the identical tool surface over `http://127.0.0.1:PORT/mcp`. CLI calls now connect to loopback — no TLS, no OAuth, no wide-area round-trip. The expensive setup happens once, in the daemon, and stays warm.

Before writing any of this: does the MCP client library already do it? [FastMCP](https://gofastmcp.com) — the Python MCP client I was already using — ships a [first-class proxy provider](https://gofastmcp.com/servers/proxy) (`create_proxy`). So the daemon is ~30 lines of glue and zero protocol code. Reuse, not reinvention.

## The trap: "stateful" doesn't mean "warm"

FastMCP exposes a `StatefulProxyClient`, and the name is a siren song. I wired the daemon with it, benchmarked, and the warm path was *no faster than direct* — ~9 seconds either way. No win at all.

The reason is a subtle scoping mismatch. `StatefulProxyClient` keeps the upstream session alive **for the duration of one incoming client session** — the right tool if each of your callers is a long-lived interactive session that wants its own isolated backend state (a browser automation session, say). But my callers are the opposite: each CLI invocation is a *brand-new, short-lived* incoming session. "Per incoming session" warmth, when every call is a new session, is no warmth at all. The proxy re-handshaked to the remote server on every call, exactly like the thing I was trying to eliminate.

This is the moment to stop guessing and read the source, not retry variants of a broken approach.

## The fix: a reentrant client, entered once

Two facts from FastMCP's internals turn out to be exactly what's needed.

**1. The client is reentrant, with a reference count.** `Client.__aenter__` doesn't blindly open a new connection — it increments a nesting counter and only actually connects on the *first* entry:

```python
# paraphrased from fastmcp/client/client.py
async def _connect(self):
    self._nesting_counter += 1
    if self._nesting_counter > 1:
        return          # already connected — reuse the live session
    ...                 # first entry: actually establish the session
```

So if I enter a client *once* and hold it open (counter = 1), every later `async with client:` bumps the counter to 2 and back to 1 on exit — the underlying session is never torn down.

**2. `create_proxy` inspects the client's connection state at construction time.** When you hand it a client that is *already connected*, it builds a "reuse this one session for all requests" factory instead of the default "open a fresh backend session per request" one. The behavior you want is latent in the library — you just have to trigger it by connecting **before** you build the proxy.

Put together, the whole daemon is this:

```python
import asyncio
from fastmcp.server import create_proxy
from slack_client import get_client, PROXY_HOST, PROXY_PORT

async def main():
    client = get_client()          # remote MCP client, with persisted OAuth
    async with client:             # enter ONCE — connect, hold the session open
        proxy = create_proxy(client, name="warm-proxy")   # sees is_connected() == True
        await proxy.run_async(transport="http", host=PROXY_HOST, port=PROXY_PORT)

asyncio.run(main())
```

The `async with client:` is the entire trick. Enter it first, and the reentrant-client-plus-connection-aware-factory combination gives you a genuine singleton warm connection with no explicit pooling, locking, or lifecycle code of your own. Restart, re-benchmark: steady-state calls dropped from ~9s to **~1.5s**.

## The honest part: I fixed the wrong-dominant cost

Here is the uncomfortable measurement, the one worth publishing precisely because it undercuts the excitement:

| path | first call | steady-state |
|---|---|---|
| warm proxy (reused loopback session) | ~4.8s | **~1.5s** |
| direct remote (fresh TLS + OAuth + WAN each) | ~4.9s | ~2.7s |

The warm proxy saves about a second per call at steady state. Real, but modest. And it does nothing for the cost that actually dominates a *one-shot CLI invocation*: **importing the MCP client library takes ~10 seconds** before a single byte crosses the network. For "paste a link, read one message," the agent shells out to a fresh Python process each time, the import tax swamps everything, and the daemon barely registers.

So I built the technically-correct fix for the wrong bottleneck. The connection *was* expensive; it just wasn't the *most* expensive thing in the path I most cared about. The warm proxy earns its keep in one clear place: many calls **inside one long-lived process** — a batch export, a search sweeping many windows — where the reused upstream amortizes across dozens of tool calls. If your callers are one-shot processes, profile the whole invocation before you optimize the network, or you'll polish the visible 20% and miss the 80%.

One tempting extension deserves an explicit warning, because I nearly recommended it myself: pointing the agent harness's *own* MCP config back at the loopback proxy. It sounds like a free win — the agent would reach a warm local endpoint instead of re-authenticating to the remote server. But it conflates two unrelated costs. The proxy fixes **connection** cost (TLS + OAuth + handshake). It does **nothing** for **context** cost — the tool schemas an MCP server injects into the model's context window, upfront, every session. That schema-in-context weight is usually the entire reason you'd strip an MCP server from an agent in the first place, and re-pointing the agent at a warm proxy brings all of it back for a marginal connection saving (the harness already holds one persistent connection per session anyway). Warm transport does not make MCP context-free. Keep the two costs separate or you'll "optimize" your way straight back to the problem you left.

## Convergent evolution as validation

Searching for prior art after the fact — the honest order is *before*, but the interjection that sent me looking came mid-build — I found [`CLIAI/slack-cli-mcp-wrapper`](https://github.com/CLIAI/slack-cli-mcp-wrapper): a different team, a different backend ([korotovsky's self-hosted server](https://github.com/korotovsky/slack-mcp-server) plus the generic [`f/mcptools`](https://github.com/f/mcptools) bridge), wrapped in Docker. Same thesis as my whole day — it even opens by quoting Anthropic's [*Code Execution with MCP*](https://www.anthropic.com/engineering/code-execution-with-mcp) (a 150k → 2k token reduction) as motivation.

Two things stood out. First, their Docker wrapper is *their* version of my warm daemon — a long-lived process that keeps the server initialized, reached over a local transport. Second, and more telling: their hardest-won debugging note describes an async cache-sync that isn't ready when a request arrives under stdio (a fresh process per call), producing a perpetual "cache is not ready yet." Their fix was to move to a persistent server. **Different backend, different bug surface, identical lesson:** async initialization plus a short-lived process is a trap, and the cure is a warm daemon. When two independent implementations converge on the same architecture to dodge the same class of bug, that's about as much validation as an architectural choice gets.

(I kept my own OAuth-based path rather than adopting theirs — their `xoxc`/`xoxd` browser-token auth is the kind a monitored corporate workspace can flag, the same reason I'd noted against `slackdump` in the [previous post]({% post_url 2026-07-10-deterministic-slack-mcp-cli %}). Convergent architecture, divergent constraints.)

## Takeaways

- **A warm local proxy in front of a remote MCP server is a reusable pattern**, not a Slack-specific hack. Anything you reach over the network with per-call auth + handshake is a candidate.
- **The reentrant-client singleton is the primitive.** Enter one client once, hold it open, and let a connection-aware proxy factory reuse it. No pool, no locks — if your library already refcounts connections, connect *before* you build the proxy.
- **"Stateful" is scoped to the caller's session, not global.** Read what the primitive actually keeps warm, and for whom, before trusting the name.
- **Connection cost and context cost are different problems.** A warm proxy makes the transport cheap; it does not make an MCP server's upfront tool-schema context cheap. If you stripped an MCP to reclaim context, do not re-point the agent at a warm proxy expecting the savings to survive — they won't.
- **Measure the whole invocation before optimizing one leg of it.** A ~1s network saving is invisible behind a ~10s import. The right fix for the wrong-dominant cost is still the wrong fix.

## Source

- [ankitg12/slack-mcp-cli](https://github.com/ankitg12/slack-mcp-cli) — the CLI and the warm-proxy daemon described here
- [FastMCP proxy provider](https://gofastmcp.com/servers/proxy) — `create_proxy` and the connection-aware client factory
- [CLIAI/slack-cli-mcp-wrapper](https://github.com/CLIAI/slack-cli-mcp-wrapper) — independent convergence on the warm-daemon pattern
- [Anthropic — Code Execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) — the token-cost argument for not loading every tool upfront
- [kb4ai/mcp-considered-suboptimal-pub-kb](https://github.com/kb4ai/mcp-considered-suboptimal-pub-kb) — a broader knowledge base on MCP's cost/quality/latency tradeoffs
