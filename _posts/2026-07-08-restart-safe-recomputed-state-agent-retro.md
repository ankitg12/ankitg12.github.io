---
layout: post
title: "Restart-safe extension state: from persisted checkpoints to pure recomputation"
date: 2026-07-08
categories: ai agents productivity
series: "AI coding agent productivity"
---

`agent-retro-omp` fires a periodic self-retrospective into a running AI coding agent session — clock-aligned or elapsed-interval, your choice ({% post_url 2026-07-03-clock-vs-elapsed-retro-triggers-omp %}). Each retro reports what happened in the segment: tool-call tally, cost, context-window growth. Restart the agent process mid-session — a crash, a Ctrl-C, an update — and until today, all of that was gone. The next retro would report only the handful of calls made since the process came back up, silently under-reporting a segment that actually spanned hours.

The first fix I reached for was wrong, and the reason it was wrong is the actual point of this post.

## The instinct: persist a checkpoint

The extension tracked two kinds of state entirely in memory:

```typescript
const activityBuffer: ActivityEntry[] = [];           // every tool call this session
const segmentStarts = new Map<string, string>();       // schedule name -> ISO start time
```

A process restart wipes both. The obvious fix is the one every engineer reaches for first: write them to disk.

```typescript
interface PersistedSegmentState {
	sessionId: string;
	segmentStarts: Record<string, string>;
	segmentCtxStartTokens: Record<string, number>;
}
```

Load on `session_start`, match against the current session id, restore if it's the same session resuming after a restart, reset otherwise. I built this. It worked. It also added a file format, a staleness-matching rule, and a restore path that needed its own tests — real complexity, for a feature that turned out not to need it.

## The question that killed it

> "Can we just compute the tool calls and info for the segment — starting from T-10 to T — would it be too much computation?"

That's the whole fix. Once you ask it, the persisted-state design falls apart on inspection:

- `segStart` for a **clock-aligned** schedule is always the last wall-clock boundary before now, minus the interval. It doesn't depend on session history at all.
- `segStart` for an **elapsed** schedule is always exactly `intervalMs` after the previous fire — including through paused or idle skips, which advance the due time from the *old* due time, not from "now."

In steady state, `segStart = fireTime - intervalMs` recovers the identical value the persisted map would have held. The only case where it diverges is a restart landing mid-window — and starting a fresh window there isn't a new edge case. It's the same "no catch-up, no backfill" rule the extension already applied to session resume. Applying it consistently to restart, too, isn't a compromise; it's the more coherent design.

```typescript
function fire(schedule: RetroSchedule): void {
	const now = new Date().toISOString();
	const segStart = new Date(new Date(now).getTime() - schedule.intervalMs).toISOString();
	// ...
}
```

No file. No session-id matching. No restore path. `segmentStarts` doesn't exist anymore.

## The buffer was the same mistake, twice

`activityBuffer` — the in-memory log of every tool call — had the identical problem, and I'd already half-solved it for a different field without noticing the pattern applied here too.

Cost tracking in this extension was never a buffer. `sumSegmentCost()` scans `sessionManager.getBranch()` — the session's own append-only transcript — for assistant messages in `[segStart, now)` and sums `usage.cost.total`. It's additive history that's already durable, on disk, unrelated to the extension's own process lifetime. Once `segStart` is derivable, tool tally can use the exact same trick:

```typescript
function getToolTally(sessionManager, segStart, segEnd) {
	const counts: Record<string, number> = {};
	let count = 0;
	for (const entry of sessionManager.getBranch()) {
		if (entry.type !== "message" || entry.message.role !== "assistant") continue;
		const ts = new Date(entry.timestamp).getTime();
		if (ts < startMs || ts >= endMs) continue;
		for (const block of entry.message.content) {
			if (block.type !== "toolCall") continue;
			counts[block.name] = (counts[block.name] ?? 0) + 1;
			count++;
		}
	}
	return { count, tally: formatTally(counts) };
}
```

`activityBuffer` and `ActivityEntry` disappear entirely. Nothing to lose on restart, because nothing is cached.

## The one I thought was a real exception

Context-usage delta felt different. `ctx.getContextUsage()` presents as a live gauge — "how much context is used *right now*" — with no obvious history to replay. I'd built a small baseline cache for it (`segmentCtxStartTokens`, reseeded on every `session_start`) and treated that as the one legitimate exception to "everything is branch-derivable."

It wasn't. Every real (non-aborted, non-error) assistant message in OMP's session log already carries:

```typescript
contextSnapshot?: { promptTokens: number; nonMessageTokens: number; lastMessageTimestamp?: number }
```

And OMP's own live gauge is computed from exactly this field — `getContextBreakdown()` walks the branch for the latest assistant message's `contextSnapshot` as ground truth, only estimating the small in-flight tail after it. The "live" number and the "historical" number are the same underlying data, just read at different points in time. So:

```typescript
function getCtxDelta(sessionManager, segStart, segEnd) {
	let startTokens, endTokens;
	for (const entry of sessionManager.getBranch()) {
		if (entry.type !== "message" || entry.message.role !== "assistant") continue;
		const snapshot = entry.message.contextSnapshot;
		if (!snapshot) continue;
		const total = snapshot.promptTokens + snapshot.nonMessageTokens;
		const ts = new Date(entry.timestamp).getTime();
		if (ts < startMs) startTokens = total;
		else if (ts <= endMs) endTokens = total;
	}
	return { startTokens, endTokens };
}
```

Baseline cache gone. `resetForSession()` no longer seeds anything for context tracking, because there's nothing left to seed. The extension now holds **zero cached or persisted per-segment state** — tool tally, cost, and context delta are all reconstructed from the branch at fire time.

## The general check

Before reaching for any persisted-state or in-memory-cache design to survive a restart, ask one question first: **is this value actually a pure function of time/config, or a live gauge with genuinely no other source — or is it already denormalized onto something durable that already exists?**

- `segStart` → pure function of fire time and interval. No storage needed.
- Cost, tool tally → additive history already on the session branch. No buffer needed.
- Context usage → *looked* live-only, but the live API itself is derived from a snapshot already sitting on each branch entry. No cache needed.

This is the same lesson [Bastian Terric's "The Snapshot Paradox"](https://docs.eventsourcingdb.io/blog/2026/03/02/the-snapshot-paradox/) makes about event-sourced systems generally: reaching for a snapshot is frequently a symptom of not re-examining whether the thing you're snapshotting is actually expensive to recompute, or whether a better-drawn boundary makes the snapshot unnecessary. Their number for a real 18-month event stream: 8,610 events, 5MB, replays in milliseconds. Sequential scan is fast; the reflex to cache is what's expensive, in code you now have to maintain and get right on restart.

## A number that looked wrong and wasn't

Once cost and context delta were both branch-derived, an obvious next metric fell out: cost per token of net context growth, in the same `$/1M` unit providers advertise pricing in.

```
Segment: 03:40 PM → 03:50 PM (10min), 51 calls (..., +41k ctx, $4.95, $120.73/1M ctx-growth)
```

$120.73/1M looks broken next to Sonnet's advertised $10/MTok output rate — a 12x gap. It isn't a bug. The two numbers measure different things:

- Provider pricing is *cost ÷ tokens sent or generated in one API call.*
- This metric is *cost ÷ (context size at segment end − context size at segment start).*

Every turn in an agentic loop re-sends the entire growing conversation as input, and re-caches it. A 51-call segment does 51 rounds of cache read/write against an already-large context, while net context *growth* that segment was only 5–41k tokens — most tool output gets consumed and doesn't survive 1:1 into the running total. [Philip Zeyliger's "Expensively Quadratic"](https://blog.exe.dev/expensively-quadratic) has the receipts: cache-read cost crosses 50% of total spend around 27,500 tokens of conversation length, and 87% by the end of a long one. The metric answers "how much did this segment's work cost per token of context it actually kept" — an efficiency signal, not a rate card. I renamed it from `$/1M ctx` to `$/1M ctx-growth` specifically so the label doesn't invite the wrong comparison.

## Verified against a real restart

The best test of "does this actually survive a restart" is an actual restart, not a simulated one. Mid-session, the OMP process bounced (`session_shutdown` → `session_start`, 11 seconds apart). The very next retro, 25 seconds after the new process came up, correctly reported the full ten-minute window — 51 calls, spanning back to before the restart. The old design would have shown a handful of calls from the new process only. It's a small thing to see work exactly as specified, on the first real occurrence, with no code path written for restart-recovery — because there's nothing to recover.

## Source

- [agent-retro-omp](https://github.com/ankitg12/agent-retro-omp) — the extension
- [Clock-time vs elapsed-time triggers for OMP's retro extension]({% post_url 2026-07-03-clock-vs-elapsed-retro-triggers-omp %}) — prior post in this series
- ["The Snapshot Paradox"](https://docs.eventsourcingdb.io/blog/2026/03/02/the-snapshot-paradox/) — the general principle
- ["Expensively Quadratic: the LLM Agent Cost Curve"](https://blog.exe.dev/expensively-quadratic) — why agentic-loop cost scales the way it does
