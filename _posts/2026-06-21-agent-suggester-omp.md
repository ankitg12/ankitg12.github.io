---
layout: post
title: "agent-suggester-omp: periodic Stoic/Gita/first-principles nudges for OMP"
date: 2026-06-21
categories: ai agents productivity
series: "AI coding agent productivity"
---

**Problem**: Long OMP sessions produce cognitive loops — "optimizing the wrong problem", attachment to results (Gita), fear vs clarity, or drifting from long-term goals. Full retros (clock-aligned or break-time) cost too much context; need lightweight 30-second reorientation nudges injected as steer messages.

**Solution**: `@ankitg12/agent-suggester` Pi extension. Loads via `omp plugin link` (not marketplace). Registers `ExtensionAPI` factory that subscribes to `session_start`, `tool_call`, `before_agent_start`, `agent_end`, `session_shutdown`. Schedules `setTimeout(fire, intervalMs)` on every session start (session-relative by default).

**Install** (ghq + link):
```bash
ghq get https://github.com/ankitg12/agent-suggester-omp
omp plugin link $(ghq list ankitg12/agent-suggester-omp)
```
Manifest in package.json:
```json
"pi": { "extensions": ["./agent-suggester.ts"] }
```

**Config** (`~/.omp/agent/agent-suggester.json`):
```json
{
  "intervalMs": 900000,
  "clockAlign": false,
  "picker": "random",
  "skipIfIdle": true,
  "wrapWith": "🔄 **Suggestion** — a 30-second pause invited\n\n{item}\n\n*If this shifts something, say so and we'll adjust. Otherwise continue — no analysis needed.*",
  "items": [
    "Are you solving the right problem, or optimizing the wrong one?",
    ...
    "This too shall pass."
  ]
}
```
- `loadConfig` validates non-empty string[] items, falls back to defaults, logs to `agent-suggester.log`.
- If absent or invalid → early return, extension idle.

**Scheduling & injection**:
- `session_start` → `toolCallCount=0`, `rotationIndex=0`, `scheduleNext()` (delay = intervalMs or nextBoundary if clockAlign).
- `fire()`: skip if paused or (skipIfIdle && toolCallCount===0); pick item, `resolveItem` (strip YAML frontmatter if `~/path`), `pi.sendUserMessage(message, {deliverAs:"steer"})`, `scheduleNext()`.
- Sleep detector: `setInterval` watching `Date.now()` delta > 2× poll → reschedule.
- `before_agent_start` (if nudgeJustFired): save `ctx.ui.getEditorText()`; `agent_end`: restore via `setEditorText`.
- Commands: `/suggester-pause`, `/resume`, `/now` (immediate random/rotate pick + steer, no timer).

**Evidence** (2026-06-21 session):
 - 01:05:35 nudge received with item[0] + exact wrapWith.
 - suggester.log: "session_start — armed timer" (no idle path).
**Why it works**: steer injection bypasses normal turn; draft save prevents editor loss; skipIfIdle avoids noise on idle segments; random vs rotate picker; 9+ curated items (Stoic, Gita, first-principles).
Repo: https://github.com/ankitg12/agent-suggester-omp
Extension source: `agent-suggester.ts` (319 LOC, no deps).