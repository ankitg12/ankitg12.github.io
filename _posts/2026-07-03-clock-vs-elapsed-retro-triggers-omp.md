---
layout: post
title: "Clock-time vs elapsed-time triggers for OMP's retro extension"
date: 2026-07-03
categories: ai agents productivity
series: "AI coding agent productivity"
---

An hourly retro fired five minutes into a session. Not because five minutes of work deserved a retrospective — because the wall clock crossed an hour boundary shortly after the session started. The retro extension was doing exactly what it was built to do. It just wasn't the right thing for an hourly cadence.

This is a follow-up to [agent-retro-omp]({% post_url 2026-06-21-agent-suggester-omp %}), the OMP extension that injects retro prompts at scheduled boundaries. It adds a second trigger mode and, in the process, surfaces a subtler bug: a naive elapsed-time timer drifts and can busy-loop under skip/pause/sleep conditions that a wall-clock timer never has to worry about.

---

## The problem with one trigger mode

`agent-retro-omp` originally had exactly one scheduling strategy: align to the next wall-clock multiple of `intervalMs`. A 10-minute mini-retro fires at `:00`, `:10`, `:20`. An hourly retro fires at `:00` on the hour.

That's the right behavior for a mini-retro — it's a metronome, and you want it ticking on a predictable cadence regardless of when you happened to start working. It's the *wrong* behavior for an hourly retro, whose entire point is to look back over a full hour of session activity. If the session starts at 2:55, the next `:00` boundary is five minutes away, and the "hourly" retro fires having seen five minutes of work, not sixty.

The fix is a second trigger mode alongside the existing one:

```json
{
  "retros": [
    { "name": "mini",   "intervalMs": 600000,  "trigger": "clock",   "prompt": "~/.claude/commands/retro.md" },
    { "name": "hourly", "intervalMs": 3600000, "trigger": "elapsed", "prompt": "~/.claude/commands/retro-long.md" }
  ]
}
```

- **`clock`** (default, unchanged) — next wall-clock multiple of `intervalMs`. Good for frequent, rhythm-setting retros.
- **`elapsed`** — fires `intervalMs` after the schedule's own last fire, or after session start for the first fire. Good for retros that need a full, uninterrupted window of activity behind them.

## The part that isn't obvious: drift and busy-loops

Adding the mode itself is a one-line branch:

```typescript
function nextFire(schedule: RetroSchedule, fromMs: number = Date.now()): number {
	return schedule.trigger === "elapsed" ? fromMs + schedule.intervalMs : nextBoundary(schedule.intervalMs);
}
```

The extension already re-arms its timer (`scheduleNext`) from more places than "a retro just fired": it also re-arms when a retro is *skipped* (session paused, or no tool activity in the segment — no point retrospecting on nothing) and when a sleep/wake watchdog detects the process was suspended and reschedules every timer from the current clock.

For `clock` mode, recomputing `nextBoundary()` from "now" on every one of those calls is correct — the next `:00` is always the next `:00`, no matter when you ask. `elapsed` mode doesn't have that property. If a re-arm recomputes `now + intervalMs` every time it's called, three things break:

1. **Drift on wake.** An hourly retro 55 minutes into its countdown, on a laptop that sleeps and wakes, gets rescheduled as "one hour from now" — the 55 minutes already elapsed are silently discarded.
2. **Busy-loop on skip.** A retro reaches its due time while the session is paused. It skips, and reschedules — but if the reschedule recomputes from "now" (which is *at* the due time, since that's when the timer fired), the new delay is ~0. It fires again immediately, skips again, reschedules again. Forever, or until the session is un-paused.
3. **No catch-up on resume**, which is actually a property you *want*: a session that resumes long after being closed should start a fresh countdown, not immediately fire a backlog of "overdue" retros.

The fix is to stop treating every re-arm the same way. There's a persisted due-time map (`nextDueAt`), and `scheduleNext` takes an explicit origin:

```typescript
function scheduleNext(schedule: RetroSchedule, origin: "fresh" | "advance" | "preserve" = "fresh"): void {
	let due: number;
	if (schedule.trigger === "elapsed") {
		const prevDue = nextDueAt.get(schedule.name);
		if (origin === "preserve" && prevDue !== undefined) due = prevDue;
		else if (origin === "advance" && prevDue !== undefined) due = prevDue + schedule.intervalMs;
		else due = nextFire(schedule);
		nextDueAt.set(schedule.name, due);
	} else {
		due = nextFire(schedule); // clock mode: always idempotent, origin is irrelevant
	}
	...
}
```

| Call site | Origin | Why |
|---|---|---|
| Session start | `fresh` | Brand-new countdown from now. No backfill for time the session wasn't open. |
| Retro actually sent (or errored) | `fresh` | The schedule completed a real cycle; start the next one from this moment. |
| Skipped — paused or no activity | `advance` | The timer legitimately reached its due time. Move the target forward by exactly one interval **from the old due**, not from "now" — otherwise repeated skips busy-loop at zero delay. |
| Wake-from-sleep re-arm | `preserve` | This re-arm happens *before* the due time is reached — it's just re-establishing a `setTimeout` after the event loop was frozen. The target itself doesn't move. |

Five call sites, three semantics, one map. The distinction that mattered most in practice was `advance` vs `preserve` — they look almost identical ("reschedule without firing") but one has to move the goalpost and the other must not.

## Verifying it without a live OMP session

The extension only really runs inside OMP's extension host, which makes the scheduling logic annoying to exercise directly. The scheduling decision itself — given a fake clock and an origin, what's the next due time — is pure and has no OMP dependencies, so it's just as easy to pull it out and run it under a fake clock in an ordinary Bun script:

```typescript
let fakeNow = 0;
const nextDueAt = new Map<string, number>();
// ...same scheduleNext logic, parameterized on a fake `now`...

// 55 minutes in, machine sleeps and wakes
fakeNow = 55 * 60_000;
const due2 = scheduleNext(hourly, "preserve");
assert(due2 === due1, "wake-from-sleep preserve does not move the due time");
```

Nine assertions — drift-on-wake, busy-loop-on-skip, fresh-reset-on-real-fire, clock-mode-ignores-origin, no-catch-up-on-new-session — catch the whole class of bug without spinning up a session. Cheap enough that it's staying in the repo as a permanent regression check.

## A second bug, found the same day: handoff leaves the clock running

Hours after shipping the fix above, an hourly retro fired reporting a 60-minute segment that spanned backward across an OMP session **handoff** — the retro's own "arc of the hour" included work from a different conversation entirely, one that had already ended.

The first hypothesis was reasonable and wrong: `agent-retro-omp` only reset its per-schedule state (`activityBuffer`, `segmentStarts`, elapsed-mode due-times) on the `session_start` lifecycle event. A handoff/resume that keeps the same extension host alive was assumed to skip `session_start` and inherit stale state — so the fix seemed to be listening for `session_switch` too, which OMP already emits for `/resume`, `/fork`, and `--continue`. That part shipped ([`agent-retro-omp@66a9c28`](https://github.com/ankitg12/agent-retro-omp/commit/66a9c28a2a3b14d3f6c57a146e8347307cf4e2fa)): the reset logic was pulled into a pure, OMP-independent `resetRetroStateForNewSession()` in `retro-state.ts`, shared by both `session_start` and a new `session_switch` handler, with a smoke test importing the real function instead of a hand-copied mirror.

It fixes `/resume`, `/fork`, and `--continue`. It does not fix `/handoff` — because `/handoff` isn't wired to either event.

Reading `AgentSession.handoff()` in OMP core directly settled it: manual and auto-triggered handoff both call the **low-level** `SessionManager.newSession()`, not the higher-level `AgentSession.newSession()`/`switchSession()` that emit `session_switch`. Handoff resets the agent in place (`agent.reset()`) and injects the carried-forward summary as a `custom_message` entry — but emits **no extension hook at all**. Worse, `session_stop` is *deliberately* skipped whenever a continuation is queued (there's a comment in the source to that effect — hook continuations there would race the handoff), and `session_shutdown` never fires because the process never exits. Every plausible hook is either silent or explicitly suppressed for this one transition.

That rules out fixing it from inside the extension. A `registerCommand("handoff", ...)` shadow doesn't work — OMP's TUI command discovery filters extension commands through a reserved-name list and skips collisions with a warning. Injecting a message from the `input` hook doesn't work either — `input` fires before slash-command dispatch, but `handleHandoffCommand()` is awaited synchronously right after, so anything an extension queues races the handoff's own `agent.reset()` with no ordering guarantee.

So the fix isn't a workaround, it's a missing hook — filed upstream as [can1357/oh-my-pi#4434](https://github.com/can1357/oh-my-pi/issues/4434): extend the existing `SessionSwitchEvent.reason` union (currently `"new" | "resume" | "fork"`) with `"handoff"`, emitted right before the old session's state is torn down, so an extension gets the one thing it actually needs — a chance to look at the *outgoing* session before it's gone, not just a signal that a new one started. If it lands, `agent-retro-omp`'s existing `session_switch` handler picks up handoff coverage for free.

Until then, the honest status is: `/resume`/`/fork`/`--continue` are fixed and tested; `/handoff` is diagnosed, documented, and reported — not patched. There wasn't a safe local fix to ship.

---

## Repo

`github.com/ankitg12/agent-retro-omp`

Config lives at `~/.omp/agent/agent-retro.json`; the extension reads it once at load, so a config edit needs a fresh session (or an extension reload) to take effect — there's no file watcher.

## Related posts

- [agent-suggester-omp: periodic Stoic/Gita/first-principles nudges for OMP]({% post_url 2026-06-21-agent-suggester-omp %}) — the sibling extension this one shares a scheduling pattern with
- [Clock-aligned retrospection in Claude Code — no WezTerm required]({% post_url 2026-06-21-clock-aligned-retro-claude-code %}) — the CC-native port of the original clock-aligned idea
- [Clock-aligned retrospection for AI coding agents](https://ankitg12.github.io/ai/agents/productivity/2026/05/06/clock-aligned-retro-for-ai-agents.html) — the original OMP version this extension descends from

---

> Generated with [Claude](https://claude.ai) (Anthropic) based on a real session with @ankitg12.
> #ai #llm #aiassisted #agenticai #softwareengineering
