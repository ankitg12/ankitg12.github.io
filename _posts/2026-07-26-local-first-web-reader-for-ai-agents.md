---
layout: post
title: "Your AI Agent Does Not Need a Hosted Reader to Turn Web Pages into Markdown"
date: 2026-07-26
categories: ai agents productivity tools privacy
series: "AI coding agent productivity"
---

Giving an agent a URL should not force a choice between raw HTML, a heavyweight browser, and sending the page through somebody else's extraction service.

I wanted one agent-facing operation:

```text
read_url("https://example.com/article") → clean Markdown
```

The implementation ended up being a small local MCP server in front of [Defuddle](https://github.com/kepano/defuddle). It keeps extraction local, reuses existing authenticated readers when appropriate, and adds almost nothing to the agent's static context.

---

## Why a URL is not yet useful context

An agent can fetch a URL, but the returned representation is often the wrong one:

- raw HTML spends context on navigation, styles, scripts, and page chrome;
- JavaScript-heavy pages may return only an application shell;
- browser automation can render the page, but is expensive when all the agent needs is prose;
- hosted reader services produce good Markdown, but private content leaves the machine;
- authenticated systems often already have a reliable local CLI, so sending them through a generic web reader duplicates authentication.

The useful abstraction is not "fetch bytes." It is **read this URL as bounded, provenance-preserving Markdown**.

## The design

The reader is a local [Model Context Protocol](https://modelcontextprotocol.io/) server exposing a single tool:

```python
from fastmcp import FastMCP

mcp = FastMCP("local-read-url")

@mcp.tool
def read_url(url: str, max_chars: int = 100_000) -> str:
    """Read an HTTP(S) URL as clean Markdown on this machine."""
    content = route_and_read(url)
    return truncate_locally(content, max_chars)
```

Behind that narrow interface is a router:

```text
                         ┌─ configured private host + known URL shape
read_url(url) ─ validate ┤       → existing authenticated local reader
                         │
                         └─ every other HTTP(S) URL
                                 → Defuddle CLI → Markdown
```

For ordinary pages the command is deliberately boring:

```bash
npm install -g defuddle
defuddle parse https://example.com/article --markdown --frontmatter
```

Defuddle was built for the Obsidian Web Clipper. Compared with a basic readability pass, it aims to preserve structures that matter in technical writing: headings, code blocks, footnotes, math, metadata, and links.

## Why route instead of installing several MCP servers?

The obvious design is one MCP server per reader. That makes the agent responsible for choosing among tools such as:

```text
read_public_page
read_private_issue
read_private_document
```

But tool choice is plumbing, not reasoning. The URL already contains enough information to route deterministically.

One `read_url` tool has three advantages:

1. **Smaller schema surface** — only one tool description enters the system prompt.
2. **Fewer agent decisions** — the model does not spend a turn choosing a reader.
3. **Centralized policy** — validation, timeout handling, provenance, and output limits live in one place.

This follows the same principle as a Unix front-end command: keep the interface stable and move backend selection into deterministic code.

## The security boundary is the hostname

A URL path is attacker-controlled. This is unsafe:

```python
if "/browse/" in url:
    return authenticated_reader(url)
```

An external URL such as `https://attacker.example/browse/PROJECT-123` now looks like a private issue URL.

The router instead requires both:

1. the parsed hostname exactly matches the hostname in the local reader's configuration; and
2. the path or query matches the expected resource shape.

```python
from urllib.parse import urlparse

parsed = urlparse(url)

if (
    parsed.hostname == configured_private_host
    and matches_private_resource_shape(parsed)
):
    return authenticated_reader(parsed)
```

Use `urlparse(url).hostname`, not substring matching or `netloc.endswith(...)`. Credentials should become reachable only after the trust boundary has been established.

The public fallback never receives credentials. It sees only the URL passed to Defuddle.

## Bound the result before it reaches the model

A clean page can still be enormous. The MCP tool accepts a bounded `max_chars` value and truncates locally:

```python
if not 1_000 <= max_chars <= 500_000:
    raise ValueError("max_chars must be between 1000 and 500000")

if len(content) > max_chars:
    omitted = len(content) - max_chars
    content = content[:max_chars]
    content += f"\n\n[Truncated locally: {omitted} characters omitted]"
```

This is important: truncating after the tool result enters the conversation is too late. The context cost has already been paid.

Each result also retains provenance:

```yaml
---
source: "https://example.com/article"
reader: "defuddle"
---
```

The agent can quote or revisit the source without guessing where extracted text came from.

## Static context cost

I measured the system prompt using the method described in [Know What Your Agent Sees: Measuring OMP's Context Footprint]({% post_url 2026-07-23-know-what-your-agent-sees %}).

| Configuration | Static context | Increment |
|---|---:|---:|
| No reader MCP | 27,118 tokens | — |
| Filtered hosted-reader MCP | 27,732 tokens | +614 |
| Local one-tool reader | 27,141 tokens | **+23** |

The large difference is not Defuddle versus a hosted service. It is **one narrow local tool schema versus a larger remote MCP catalog**.

The lesson generalizes: tool descriptions are part of the prompt. Installing an MCP server has a recurring context cost even when none of its tools are called.

## What I evaluated first

This was not a reason to build a web extractor. Mature extractors already exist.

| Option | Strength | Why I did not use it unchanged |
|---|---|---|
| [Defuddle Fetch MCP](https://github.com/domdomegg/defuddle-fetch-mcp-server) | Small MCP wrapper around Defuddle | Could not reuse existing authenticated local readers |
| [Official Fetch MCP](https://github.com/modelcontextprotocol/servers/tree/main/src/fetch) | Established reference implementation | Extraction was less structured for the technical page I tested |
| [Fetcher MCP](https://github.com/jae-jae/fetcher-mcp) | Playwright handles rendered JavaScript pages | Heavier than needed for the common article path |
| [Jina Reader](https://jina.ai/reader/) | Excellent URL-to-Markdown service | Hosted extraction was the wrong default for potentially private URLs |
| Defuddle CLI | Strong local Markdown extraction | Chosen as the public-web backend |

The custom part is only the policy router. Content extraction remains somebody else's well-tested problem.

## Verification

I exercised four properties rather than merely checking that the server started:

1. **Public extraction** — a technical article became 7,402 characters of Markdown with headings, links, and fenced Lua intact.
2. **Authenticated routing** — known private URL shapes reached their existing local readers.
3. **Host isolation** — the same private-looking path on an external hostname stayed on the public route.
4. **Fresh-session discovery** — a new agent session connected to the MCP server and exposed only `read_url`.

That third check is the one most likely to be skipped. It is also the one that turns a convenient router into a defensible security boundary.

## The pattern

The reusable pattern is larger than web reading:

```text
one agent-facing capability
        ↓
deterministic local router
        ↓
existing specialized backends
```

Agents should reason about the task, not about which of five plumbing tools happens to understand a URL. Keep the interface narrow, route with ordinary code, preserve provenance, and put limits at the boundary.

## Sources

- [Defuddle](https://github.com/kepano/defuddle)
- [FastMCP](https://github.com/PrefectHQ/fastmcp)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [Official MCP Fetch server](https://github.com/modelcontextprotocol/servers/tree/main/src/fetch)
- [Defuddle Fetch MCP](https://github.com/domdomegg/defuddle-fetch-mcp-server)
- [Fetcher MCP](https://github.com/jae-jae/fetcher-mcp)
- [Jina Reader](https://jina.ai/reader/)
