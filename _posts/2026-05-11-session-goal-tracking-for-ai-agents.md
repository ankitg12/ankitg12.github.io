---
layout: post
title: "Per-session goal tracking for AI coding agents"
date: 2026-05-11
categories: ai agents productivity
series: "AI coding agent productivity"
---

Clock-aligned retros tell you whether you worked well. They don't tell you whether you worked on the right thing.

Goal drift is the failure mode no one talks about: the agent is busy, productive, generating output — but it left the declared goal behind three turns ago and nobody noticed until the session ended. By then, the context is spent.

This post adds a session goal layer to the [clock-aligned retrospection framework]({% post_url 2026-05-06-clock-aligned-retro-for-ai-agents %}). Every automatic retro now opens with a verbatim goal read and a three-way verdict: **ON TRACK / DRIFTED / DONE**.

See also: [Break-time retrospection]({% post_url 2026-04-27-break-time-retro-for-ai-agents %}), [Personal Agentic Process]({% post_url 2026-05-06-personal-agentic-process %}).

---

## The problem

The `agent-retro` extension fires retro prompts at clock boundaries. Those prompts ask: was the work good? They never asked: was it the right work?

The gap: five retros could fire in a two-hour session, each reporting "no backtracking, good engineering" — while the agent silently pivots from the declared goal to an adjacent rabbit hole that looked interesting at the time.

The fix is simple: every retro reads the session goal and states the verdict. If the agent can't say "ON TRACK" without hesitation, that's the signal.

---

## Design

### Goal file

Each session gets one goal file, co-located with its JSONL session file:

```
~/.omp/agent/sessions/<cwd>/<uuid>.goal.md
```

Format:

```yaml
goal: <one sentence>
started: YYYY-MM-DD HH:MM
constraint: <none | X hours | hard deadline HH:MM>
context: <continuing: <what> | fresh start>
```

Written once at session start via `/session-start` or PAP step 4. Also archived to `~/sessions/YYYY/MM-Month/YYYY-MM-DD_HH-MM-SS - <slug>.md` for the permanent record.

### Why UUID-keyed, not a single shared file

The naive design is one file: `~/.claude/session-goal.md`. It works until you run two OMP sessions in parallel — both write to the same file, last writer wins, retros in both sessions read the wrong goal.

OMP session IDs are `Bun.randomUUIDv7()` — time-ordered, unique, stable. The UUID doesn't change when you `/rename` a session (rename only updates the title field in the JSONL header). So `<uuid>.goal.md` is collision-free and survives renames.

The file accumulates alongside the JSONL. No cleanup needed at `close-session` — it gets pruned when the JSONL is pruned.

### Retro ladder integration

Every level of the retro ladder reads the goal file:

| Level | Cadence | Goal check |
|---|---|---|
| `retro-mini` | per break | 3-line pulse: goal verbatim + ON TRACK / DRIFTED / DONE |
| `retro` | per break (full) | §0 full alignment: drift cause + next-segment intent |
| `retro-long` | hourly | §2 verdict + time-constraint pacing |
| `close-session` §3.5 | end of session | ACHIEVED / PARTIALLY ACHIEVED / MISSED / SUPERSEDED → archived to Logseq |

---

## Extension: content injection

The `agent-retro-omp` extension already injects retro prompts via `sendUserMessage`. Adding goal content to the same message costs nothing and saves the agent a file-read tool call on every retro.

At `session_start`, the extension captures the session file path:

```typescript
pi.on("session_start", (_event, ctx) => {
  sessionFile = ctx.sessionManager.getSessionFile() ?? undefined;
});
```

At each `fire()`, it reads the goal file and embeds the content as HTML comments before the retro prompt:

```typescript
const goalFile    = sessionFile?.replace(/\.jsonl$/, ".goal.md");
const goalContent = goalFile && existsSync(goalFile)
  ? readFileSync(goalFile, "utf-8").trim()
  : "NOT SET — run /session-start to declare one.";

const sessionCtx = sessionFile
  ? `<!-- session-file: ${sessionFile} -->\n` +
    `<!-- session-goal: ${goalFile} -->\n` +
    `<!-- session-goal-content:\n${goalContent}\n-->\n\n`
  : `<!-- session-goal-content:\n${goalContent}\n-->\n\n`;
```

The retro message arrives pre-loaded with the goal. The agent reads `<!-- session-goal-content -->` directly and proceeds — no bash, no file read, one less tool call per 10-minute boundary.

The path is still embedded in `<!-- session-goal: PATH -->` for cases where the agent needs to write or update the goal.

---

## Two-tier fallback

Auto-fired retros (from the extension) always have the `<!-- session-goal-content -->` comment. Manual invocations (`/retro`, `/retro-mini`) don't. The retro prompts handle both:

1. **Comment present** → read content directly from `<!-- session-goal-content -->`
2. **Comment absent** → `ls -t ~/.omp/agent/sessions/*/*.goal.md 2>/dev/null | head -1` then read that file

If both fail: goal is undeclared — flag it as a process gap and prompt `/session-start`.

---

## What didn't work

**`~/.claude/session-goal.md` as a live pointer** — the original design had the extension point at a fixed path and all retro commands read from there. It works for a single session but fails silently with parallel sessions. The smell: the retro-long for session A reports the goal of session B, which wrote the file more recently. UUID-keyed files eliminate the class of bug.

**`session-goal-$OMP_SESSION_NAME.md`** — requires the user to set an env var before launching OMP. Env vars don't survive across bash tool calls (each call spawns a new MSYS2 shell) unless exported in the persistent shell session. More importantly, they require manual discipline. The UUID file requires no env var — the extension derives the path from `ctx.sessionManager.getSessionFile()` which is always available.

---

## The verdict in practice

When a retro fires and the goal is present, step 0 produces something like:

```
Goal: Build the per-session goal tracking framework
Verdict: ON TRACK — currently wiring the extension injection
Next segment: commit agent-retro-omp, update retro commands, push
```

When the goal is absent (file not written yet):

```
Goal: NOT SET — run /session-start to declare one.
```

The "NOT SET" signal embedded in the message means the agent flags it immediately, in the retro turn, without an extra round-trip.

---

## OMP internals used

- `ctx.sessionManager.getSessionFile()` — returns the JSONL path for the current session
- `ctx.sessionManager.getSessionId()` — returns the UUID (stable across `/rename`)
- `readFileSync` / `existsSync` from Node `fs` — already imported in the extension
- `session_start` event — fires once when OMP loads the session, before any user turn

---

## Source

- [agent-retro-omp](https://github.com/ankitg12/agent-retro-omp) — the extension (`agent-retro.ts`, ~280 lines of TypeScript)
- [claude-config](https://github.com/ankitg12/claude-config) — retro prompts and slash commands
- [agentpatterns.ai: Goal Recitation](https://agentpatterns.ai/context-engineering/goal-recitation/) — the named pattern this implements
- [agentpatterns.ai: Agent Loop Middleware](https://agentpatterns.ai/agent-design/agent-loop-middleware/) — why harnesses should inject resolved content, not file pointers

---

*This post was written during the session that built the framework. The retro fired six times. Goal verdict at each fire: ON TRACK.*
