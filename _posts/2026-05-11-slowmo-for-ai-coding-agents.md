---
layout: post
title: "Slow-motion mode for AI coding agents"
date: 2026-05-11
categories: ai agents productivity
series: "AI coding agent productivity"
---

AI agents fire tool calls the instant they decide to. That's usually fine. But when you're debugging a complex refactor, or the agent is about to run a destructive bash command, "the instant it decides to" is often too fast for you to read what it's about to do, let alone intervene.

This is a 60-line [OMP](https://github.com/can1357/oh-my-pi) extension that inserts a configurable pause before each tool call — with a visible countdown and a Ctrl+C abort path.

---

## The problem

The cognitive mismatch: the agent processes tool results at LLM speed (milliseconds); you process them at human speed (seconds). In a fast session, fifteen tool calls fire and complete before you've read the first one. By the time you notice something looks wrong, four more tools have already run on top of it.

Claude Code has no built-in throttle. There's no way to say "pause 3 seconds before each bash call so I can read it." There's no abort path short of killing the whole process.

---

## What it does

Before each tool call (except excluded fast lookups), the extension:

1. Prints `[slowmo] bash — 3s pause — Ctrl+C to abort` to the status bar
2. Sleeps for the configured delay (default: 3 000 ms)
3. Lets the tool execute normally if the delay completes
4. Aborts the entire agent turn if you hit Ctrl+C during the pause

Fast read-only tools (`read`, `find`, `search`, `lsp`) are excluded by default — no noise from lookups, only weight on mutations and side effects.

---

## How it works

OMP's `tool_call` event fires **pre-execution** and awaits its handler. An async sleep in the handler reliably holds up tool dispatch:

```typescript
pi.on("tool_call", async (event, ctx) => {
  if (!config.enabled) return;
  if (config.excludeTools.includes(event.tool.name)) return;

  const label = `[slowmo] ${event.tool.name} — ${config.delayMs / 1000}s pause — Ctrl+C to abort`;
  ctx.ui.setStatus("slowmo", label);

  await sleep(config.delayMs);

  ctx.ui.setStatus("slowmo", "");
});
```

`sleep` is a plain `Promise<void>` wrapper around `setTimeout`. Because the handler is `async` and OMP awaits it, no tool runs until the promise resolves.

Ctrl+C during the pause cancels the turn because OMP propagates the abort signal to all pending awaits. The tool call never fires.

---

## Config

Stored at `~/.omp/agent/agent-slowmo.json`:

```json
{
  "delayMs": 3000,
  "enabled": true,
  "excludeTools": ["read", "find", "search", "lsp"],
  "debug": false
}
```

The extension reads this at load time. Slash commands mutate the in-memory state for the session without touching the file — so `enabled: true` in the file means always-on by default, but `/slowmo-off` kills it for the current session without permanently changing behavior.

---

## Slash commands

| Command | Effect |
|---|---|
| `/slowmo-off` | Disable for this session |
| `/slowmo-on` | Re-enable for this session |
| `/slowmo-set 5000` | Set 5 s delay and enable |
| `/slowmo-status` | Print current state |

These are OMP slash command handlers registered via `pi.registerCommand`. They write to the same in-memory config object the `tool_call` handler reads — no file I/O on the hot path.

---

## Coexistence with agent-retro

Both extensions hook `tool_call`. `agent-retro` observes post-execution (it records metadata after the tool completes). `agent-slowmo` delays pre-execution. They don't interfere: handler order matches extension load order in `config.yml`, and both run before the tool fires.

```yaml
extensions:
  - ~/repos/github.com/ankitg12/agent-slowmo-omp
  - ~/repos/github.com/ankitg12/agent-retro-omp
```

Result: slowmo fires first (pause), then the tool runs, then retro fires (observation).

---

## What didn't work

**A confirm dialog instead of a timed delay.** OMP exposes `ctx.ui.confirm()` — a yes/no prompt the agent suspends on. The intent was: show the tool call, wait for explicit approval. Problem: `confirm()` in a `tool_call` handler deadlocks in some OMP versions when the agent is mid-turn. A timed sleep with a Ctrl+C escape is simpler, always works, and requires no keypress for the 95% of calls you want to let through.

**`pi.on("before_tool_call")`.** That event doesn't exist in the current OMP API. `tool_call` is the correct pre-exec hook — it fires before the tool runs if and only if the handler awaits.

---

## When to use it

- **Debugging sessions** where you want to read each tool call before it executes
- **Destructive operations** (large refactors, schema migrations, file deletions)
- **Learning a new codebase** with an agent — watch the calls, build a mental model
- **Any session where the agent is faster than you want it to be**

Flip it off (`/slowmo-off`) when you've verified the agent is on track and want it to run at full speed.

---

## Source

- [agent-slowmo-omp](https://github.com/ankitg12/agent-slowmo-omp) — the extension (`agent-slowmo.ts`, ~60 lines of TypeScript)
- [Timestamps post]({% post_url 2026-04-27-timestamps-for-ai-coding-agents %}) — the extension this pairs with
- [Personal Agentic Process]({% post_url 2026-05-06-personal-agentic-process %}) — the broader framework

---

*Written in a slowmo session. Every bash call waited 3 seconds. Caught one wrong path substitution before it ran.*
