---
layout: post
title: "Clock-aligned retrospection for AI coding agents"
date: 2026-05-06
categories: ai agents productivity
series: "AI coding agent productivity"
---

AI coding agents run for hours without pausing to reflect. Long sessions drift — the agent repeats approaches, accumulates dead ends, loses sight of the original goal. A periodic check-in fixes this.

This post describes `agent-retro`: an [OMP](https://github.com/can1357/oh-my-pi) extension that injects retrospection prompts at wall-clock boundaries. No external tools required.

See also: [Adding timestamps to AI coding agent conversations]({% post_url 2026-04-27-timestamps-for-ai-coding-agents %}).

---

## What it does

Two configurable schedules fire automatically throughout the session:

| Name | Default | Prompt |
|---|---|---|
| `mini` | every 10 min | brief segment review |
| `hourly` | every 60 min | full hour retrospection |

When a boundary fires, the agent receives the retro prompt as a message and responds inline — no user intervention needed. If there was no activity in the segment, the retro is skipped silently.

---

## Clock alignment

Boundaries align to the local wall clock, not to session start. Mini fires at :00, :10, :20... Hourly fires at the top of each hour. A session starting at 4:58 PM will fire its first mini at 5:00 PM, covering the two minutes of activity before it — not 10 minutes from now.

This makes retros predictable. You know when to expect them. Multiple concurrent sessions fire independently at the same boundaries.

---

## The retro prompt

The prompt is a markdown file — point it at whatever you want the agent to reflect on. A minimal example:

```markdown
You are running a break-time retrospection. Answer each question:

1. **Done**: What was accomplished? Did we achieve the goal?
2. **Stuck?**: Any repeated attempts or backtracking?
3. **Don't reinvent the wheel**: Did you check for existing tools before writing code?
4. **Next**: What is the highest-value next action?
```

The extension prepends an activity summary before the prompt:

```
Segment: 04:10 PM → 04:20 PM (10min), 31 calls (18 bash, 8 edit, 3 read, 2 web_search)
```

---

## How it works

Three mechanisms combined:

**Clock-aligned scheduling** — `setTimeout` to the next boundary. On `session_start`, timers arm from the *previous* boundary so the first fire covers a full window even if the session started mid-interval.

**Activity tracking** — every `tool_call` event appends `{ timestamp, toolName, intent }` to an in-memory buffer. On fire, the buffer is sliced to the segment window, formatted into the summary, and cleared.

**Delivery** — `sendMessage` with `triggerTurn: true`, no `deliverAs`. When idle, this fires an immediate agent turn. When the agent is streaming, it steers between tool calls.

**Sleep/resume** — a 10-second poll detects when `Date.now()` jumps beyond the expected tick (laptop wake from sleep) and reschedules stale timers.

---

## What didn't work

**`deliverAs: "nextTurn"`** — looks right in the docs, silently broken. The `nextTurn` branch in OMP's `sendCustomMessage()` parks messages in `_pendingNextTurnMessages` and bypasses `triggerTurn` entirely, even when both are set. The message sits until the user manually sends input. Drop `deliverAs` entirely and use only `triggerTurn: true`.

**`omp plugin install git:...`** — OMP's plugin installer doesn't handle git URLs. Use a local path in `config.yml` instead.

---

## Setup

```bash
# Clone via ghq
ghq get github.com/ankitg12/agent-retro-omp

# Add to ~/.omp/agent/config.yml
extensions:
  - ~/repos/github.com/ankitg12/agent-retro-omp

# Configure (~/.omp/agent/agent-retro.json)
{
  "retros": [
    { "name": "mini",   "intervalMs": 600000,  "prompt": "~/.claude/commands/retro.md" },
    { "name": "hourly", "intervalMs": 3600000, "prompt": "~/.claude/commands/retro-long.md" }
  ]
}
```

Restart OMP. Check `~/.omp/agent/agent-retro.log` — you should see `session_start — arming 2 retro(s)` and the next scheduled fire times.

---

## Source

- [agent-retro-omp](https://github.com/ankitg12/agent-retro-omp) — OMP extension, ~200 lines of TypeScript

---

*This post was written during a session where the retro was running. It fired three times.*
