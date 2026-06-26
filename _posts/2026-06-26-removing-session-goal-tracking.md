---
layout: post
title: "Why I removed per-session goal tracking (six weeks later)"
date: 2026-06-26
categories: ai agents productivity
series: "AI coding agent productivity"
---

Six weeks ago I [built per-session goal tracking]({% post_url 2026-05-11-session-goal-tracking-for-ai-agents %}) for my OMP retro extension. The idea: each session gets a goal file, the extension injects it as an HTML comment before every retro prompt, the agent reads it and states a three-way verdict — **ON TRACK / DRIFTED / DONE**.

This morning I removed it.

Not because the implementation broke. Because after six weeks, I had eight or more consecutive sessions with this in every retro:

```
<!-- session-goal-content:
NOT SET — run /session-start to declare one.
-->
```

The retro agent dutifully flagged it as a process miss every ten minutes. By the fourth session in a row, the flag had become noise. By the eighth, it was generating guilt without generating clarity.

---

## What went wrong

The design assumed sessions start fresh, with a clean goal in mind, before the first tool call. That's not how sessions actually start.

Most sessions begin mid-thought: "where was I?" followed immediately by reading a file or checking git status. The goal exists, but it's implicit — a continuation of yesterday's blocker or a quick fix that surfaced in the morning. Making it explicit requires a `/session-start` command, a one-sentence goal, a frontmatter write. That's 30 seconds of overhead on a good day. When you're in flow and open a session to chase a specific thing, that 30 seconds is enough friction to skip it.

And once you skip it once, you skip it twice. After eight sessions, the habit is dead.

The deeper problem: the retro questions that actually caught drift were already there. "Done — one concrete sentence. Was it worth the segment?" does the goal-check implicitly. If the agent can answer that without mentioning the original objective, you've already drifted — and you'll see it. The explicit goal file was redundant for everything except the case where I actually declared a goal, which I stopped doing.

---

## What survived

The **daily goal** (`~/.omp/agent/daily-goal.md`) is different. It covers a full day's objectives — "unblock Leni SSDK ping test, deepen SONiC architecture understanding" — and is read at session start, not declared interactively. It survives because:

1. It's written once per day, not once per session.
2. It's read by the extension, not prompted from the user.
3. A day has 2-4 sessions; a session has 10-30 retros. The per-day granularity matches the actual cadence of intention-setting.

The **daily close goal verdict** also survives. At day's end, looking back and assigning ✅ / 🔶 / ❌ to each daily goal costs nothing — the session is closed, there's no friction, and the retrospective is honest. The problem was per-session overhead, not per-day reflection.

---

## The change

Three deletions in `agent-retro.ts`:

```typescript
// removed: goalFile derivation
const goalFile = sessionFile?.replace(/\.jsonl$/, ".goal.md");
const goalContent = goalFile && existsSync(goalFile) ? ... : "NOT SET...";

// removed: goal injection into retro message  
const sessionCtx = idTag +
  `<!-- session-goal-content:\n${goalContent}\n-->\n` +
  (goalFile ? `<!-- session-goal-path: ${goalFile} -->\n` : "");
```

The retro message now opens with only the session UUID tag. No goal read, no process-miss flag, no guilt.

Section 3.5 ("Session Goal Outcome") removed from the `close-session` skill. The `/session-start` command still exists but is no longer wired into anything — it can write the goal file, but nothing reads it.

---

## The principle

A process ritual that generates guilt without generating clarity should be removed.

Guilt is a signal: something expected didn't happen. But if what was expected requires friction that compounds over time, the signal is pointing at the wrong thing. The retro was telling me "you didn't declare a goal" when the real message was "the habit of declaring a goal before every session doesn't fit how you actually work."

The fix isn't discipline. It's removing the ritual and keeping the outcome — drift detection — through mechanisms that don't require upfront declaration. The "Done" question in the retro already does this. The daily close verdict does this at day granularity. The per-session layer was noise.

---

## Takeaway

If you implement something from [the original post]({% post_url 2026-05-11-session-goal-tracking-for-ai-agents %}): watch your NOT SET rate over the first two weeks. If it's above 50%, the habit won't form. Either reduce the friction (auto-detect goal from journal, don't require `/session-start`) or drop the per-session layer and work at daily granularity instead.

The auto-detect path is worth exploring. Reading today's journal, extracting the goals block, and injecting it without any user action would remove the friction entirely. I haven't built that yet.

---

## Source

- [agent-retro-omp](https://github.com/ankitg12/agent-retro-omp) — the extension, post-removal (`agent-retro.ts`, commit `5b30bfd`)
- [Per-session goal tracking]({% post_url 2026-05-11-session-goal-tracking-for-ai-agents %}) — the original post
