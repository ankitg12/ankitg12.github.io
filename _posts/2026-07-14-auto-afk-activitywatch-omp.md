---
layout: post
title: "Auto-AFK: detecting real-world idle instead of typing /afk"
date: 2026-07-14
categories: ai agents productivity
series: "AI coding agent productivity"
---

The [AFK toggle]({% post_url 2026-05-27-afk-toggle-for-ai-coding-agents %}) I built in May required remembering to type `/afk` before stepping away. Half the time I forgot — I'd just get up, and the agent would sit idle waiting for a prompt that never came. The fix isn't a better reminder, it's removing the step: detect that I've actually left the desk and flip the switch automatically.

---

## The missing piece

`/afk` and `/back` already did everything needed once engaged — status bar indicator, autonomy prompt injection, guard against double-engagement. What was missing was the *trigger*. Manual commands only fire if you remember to fire them, and "remember to tell the computer you're about to forget about the computer" is a bad UX pattern.

The obvious signal already existed on the machine: [ActivityWatch](https://activitywatch.net/), which I already run for time tracking, ships `aw-watcher-afk` — a small watcher that samples mouse/keyboard activity and writes `afk`/`not-afk` events to a local bucket, queryable over a REST API on `localhost:5600`. No new dependency, no new permission prompt, just a bucket that was already being written to.

---

## What it does

Same `/afk` / `/back` commands as before, plus:

- A 15-second poll of ActivityWatch's AFK bucket
- After a **sustained 2-minute** AFK reading (not the first tick — a 2-minute debounce avoids auto-engaging because you looked away to read a Slack notification for 20 seconds), it calls the exact same `engage()` path as typing `/afk`
- The moment activity resumes, it calls the exact same `disengage()` path as `/back` — but **only if the current engagement was auto-triggered**. If you typed `/afk` yourself and then, say, walk to get coffee and the AFK watcher fires again, it does *not* silently disengage a session you engaged on purpose.

```typescript
function engage(ctx: ExtensionContext, viaAuto: boolean): void {
  if (afkActive) return;
  afkActive = true;
  autoEngaged = viaAuto;
  ctx.ui.setStatus(STATUS_KEY, viaAuto ? "AFK 🔴 auto" : "AFK 🔴");
  ctx.ui.notify(`[afk] engaged${viaAuto ? " (auto)" : ""}`, "info");
  pi.sendUserMessage(buildAfkPrompt(sessionId), { deliverAs: "followUp" });
}

async function poll(ctx: ExtensionContext): Promise<void> {
  const status = await fetchLatestAfkStatus(config);
  if (status === "afk") {
    afkSince ??= Date.now();
    if (!afkActive && Date.now() - afkSince >= config.debounceMs) {
      engage(ctx, true);
    }
    return;
  }
  afkSince = undefined;
  if (afkActive && autoEngaged) disengage(ctx);
}
```

The manual and automatic paths share `engage()`/`disengage()` — there's exactly one code path that does the actual work, and two ways to trigger it.

---

## Bucket auto-discovery, not a hardcoded name

ActivityWatch names buckets `<watcher>_<hostname>` — e.g. `aw-watcher-afk_MYMACHINE`. Hardcoding that string ties the extension to one machine. Instead, the extension queries `GET /api/0/buckets/` once, finds whichever bucket has `type: "afkstatus"` (the type is stable; the name isn't), and caches the result for the life of the process:

```typescript
let discoveredBucket: string | undefined;

async function discoverAfkBucket(baseUrl: string): Promise<string | undefined> {
  if (discoveredBucket) return discoveredBucket;
  const res = await fetch(`${baseUrl}/`);
  const buckets: unknown = await res.json();
  for (const value of Object.values(buckets)) {
    if (isAwBucketInfo(value) && value.type === "afkstatus") {
      discoveredBucket = value.id;
      return discoveredBucket;
    }
  }
}
```

One request per process lifetime, not one per poll — at a 15-second poll interval that's the difference between 1 request/day and ~5,760.

---

## The mistake worth writing down

I built the first version of this from scratch — new repo, new poller, new prompt injection — without checking whether `agent-afk-omp` already existed. It did: five commits, already deployed in my OMP config, with the exact `/afk`/`/back` mechanism this post's predecessor documented. I overwrote its `agent-afk.ts` in place before noticing.

The recovery: `git show HEAD:agent-afk.ts` to pull the original content back, then a manual merge — keep every original function and command signature untouched, add the poller as a new trigger path into the *same* `engage`/`disengage` functions rather than replacing them. Nothing about the original interface changed; `/afk` and `/back` behave identically to how they did in May.

The actual lesson isn't "check git log before writing code" (obviously true, not interesting). It's that **"build X" and "add X to what already exists" are different requests, and the difference only shows up if you check for existing work before deciding which one you're doing.** I had a note (`~/Notes/omp-extensions.md`) that would have answered this in one line and didn't read it first.

---

## Source

- [agent-afk-omp](https://github.com/ankitg12/agent-afk-omp) — the extension, now with auto-detection
- [AFK toggle for AI coding agents]({% post_url 2026-05-27-afk-toggle-for-ai-coding-agents %}) — the original manual version
- [ActivityWatch](https://activitywatch.net/) — the AFK-detection source, already running for time tracking
