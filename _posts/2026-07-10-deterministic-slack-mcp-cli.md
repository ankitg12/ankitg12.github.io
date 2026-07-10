---
layout: post
title: "When your AI agent's Slack search forgets the link: building a deterministic MCP CLI"
date: 2026-07-10
categories: ai agents productivity
series: "AI coding agent productivity"
---

I asked my coding agent to search Slack for something, and it gave me a clean summary — channel, author, snippet — with no link back to the actual message. I had to reopen Slack and search again by hand to find what it had just found for me.

The MCP tool call itself wasn't broken. The agent's *behavior* was inconsistent.

## Root cause: a default that silently degrades

The official Slack MCP server (the one bundled with most coding-agent harnesses today) exposes a `response_format` parameter on its search/read tools: `detailed` or `concise`. Only `detailed` includes a `Permalink` field per result. `concise` drops it entirely — no field, no fallback, nothing to recover from after the fact.

An agent choosing `concise` for a quick scan, or summarizing a `detailed` result and dropping the link along the way, produces the exact same user-visible outcome: an answer with no way to follow up. Neither is a bug in the strict sense. Both are the model's judgment call not matching what I actually needed.

The first fix was cheap: a rule, repeated everywhere the agent might touch Slack search —

1. Default to `detailed`, never `concise`, when the user might want to follow up.
2. Every cited message carries its permalink into the final answer, no exceptions.
3. If `concise` was already used and a link is needed, re-fetch that one message in `detailed` mode.

This is a *prompt-layer* fix. It works as well as the model's instruction-following does — which is usually fine, but "usually" isn't "always."

## When prompt-layer fixes aren't enough

For anything that has to behave the same way every single time — a cron job, a scripted export, a background pull with no human reading the intermediate reasoning — a rule the model *might* follow isn't good enough. That's the line where you stop prompting and start writing code.

Before writing any code: is there already a tool that wraps an MCP server from a plain script, without hand-rolling JSON-RPC and OAuth? Yes — [FastMCP](https://gofastmcp.com), the de facto Python client library for MCP. It handles both local (stdio) and remote (HTTP + OAuth 2.1) servers, and persists tokens through a pluggable key-value backend. No custom protocol code needed.

There's also a well-established personal-Slack-archival tool, [slackdump](https://github.com/rusq/slackdump) (2,600+ stars, multi-year history) — but it authenticates by extracting live browser session cookies (`xoxc`/`xoxd`), which its own README flags as something that can trigger enterprise Slack's automated-access alerts. On a monitored corporate workspace, official OAuth — even with the friction below — is the safer default. Worth naming explicitly: the more popular tool isn't always the right tool for your constraints.

## Two real bugs, not hypothetical ones

Building the wrapper surfaced two server-side quirks that no amount of reading documentation would have caught — both needed a live probe to find.

**1. OAuth's redirect_uri isn't arbitrary.** FastMCP's client tries Dynamic Client Registration by default, which the hosted Slack MCP server doesn't support (a documented, widely-hit issue). The workaround — FastMCP's "Static Client Registration," reusing a `client_id` your coding harness's own Slack integration already has consent for — worked immediately for listing tools. It failed at the actual browser-consent step with:

```
redirect_uri did not match any configured URIs
```

Slack's OAuth app registration allows only a fixed, pre-registered `redirect_uri`. A client library defaulting to "pick any free localhost port" will never match that allowlist. The fix: find the exact callback port your existing MCP client already uses for that `client_id` (check its own config), and pin `callback_port` to match.

**2. Search pagination has a much lower ceiling than the API's own limit.** Slack's search API allows up to ~10,000 results overall. The MCP server's cursor pagination, independent of that, raises `page_limit_exceeded` after exactly 20 pages — roughly 400 messages — per continuous search session. Trying to page through years of history in one query silently walls off at a few hundred messages, not the number you'd expect from Slack's own documented limits.

The fix is to stop treating "search" as one continuous cursor walk and instead window it: run the same query repeatedly across fixed date ranges (`before`/`after` on the message timestamp), each range getting its own fresh cursor budget, then merge and deduplicate the results by `(channel, timestamp)`. This is the same shape of workaround Slack's own compliance/Discovery tooling uses at a different layer — enumerate breadth (channels or date windows) instead of depth (one exhaustive cursor).

## A cheap lesson about long-running scripts

The export mode runs many of these windowed searches in sequence and can take a while for a large time range. The first version wrote its output JSON only once, at the very end of the loop. When I killed the process mid-run to change its scope, everything it had already fetched — correctly parsed, ready to save — was gone. Nothing had touched disk yet.

The fix is a five-line change with an outsized payoff: write (or rewrite) the output file after *every* window, not just at the end, with a `"complete": false/true` flag so a reader can tell whether a run finished or was cut short. Any script that loops over paginated or windowed work and only persists results at completion has this same failure mode waiting for the first `Ctrl-C`.

```python
def _write_output(path, records, complete):
    path.write_text(json.dumps({
        "count": len(records),
        "complete": complete,
        "messages": sorted(records.values(), key=lambda r: r["ts"], reverse=True),
    }, indent=2))

# after every window, not just at the end:
_write_output(out_path, all_records, complete=False)
...
_write_output(out_path, all_records, complete=True)
```

## Two paths, kept deliberately separate

The end state isn't "replace the agent's Slack tool with a script." It's two paths for two different jobs:

| | Agent-facing MCP tool calls | Deterministic CLI |
|---|---|---|
| Use for | conversational, exploratory, one-off | scripted, scheduled, must-be-consistent |
| Permalink guarantee | depends on the model following the rule | guaranteed by construction |
| Covers | full surface (post, react, canvases, ...) | search / read / export, growing |

A CLI that only covers 20% of what the conversational tool can do isn't a replacement — it's a different tool for a different reliability requirement, and it's fine (expected, even) for it to grow incrementally as new deterministic use cases show up.

## Source

- [ankitg12/slack-mcp-cli](https://github.com/ankitg12/slack-mcp-cli) — the wrapper described above
- [FastMCP](https://gofastmcp.com) — the MCP client library it's built on
- [Slack's official plan for self-serve export](https://slack.com/help/articles/204897248-Guide-to-Slack-import-and-export-tools) — confirms full history export is org-owner-only, which is why the windowed-search workaround exists at all
- [rusq/slackdump](https://github.com/rusq/slackdump) — the established alternative, and why its auth model didn't fit here
