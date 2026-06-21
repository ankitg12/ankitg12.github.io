---
layout: post
title: "Clock-aligned retrospection in Claude Code — no WezTerm required"
date: 2026-06-21
categories: ai agents productivity
series: "AI coding agent productivity"
---

Every 10 minutes, Claude Code stops mid-tool-call and runs a retrospective. Not because I typed anything. Not because I set a timer. A hook did it.

This is the Claude Code port of the [clock-aligned retro I built for OMP](https://ankitg12.github.io/ai/agents/productivity/2026/05/06/clock-aligned-retro-for-ai-agents.html). That version used WezTerm's `send-text` API to inject a slash command into the terminal. This version uses CC's native hook system — three hooks, one Python script, zero terminal dependencies.

---

## The problem it solves

AI coding agents drift. You start with a goal, the agent hits a snag, you redirect, and 40 minutes later you're debugging something three levels removed from what you opened the session for. The agent has no mechanism to notice this.

Periodic retrospection forces a pause: what did we just do, was it worth it, are we still on goal? The key insight from the OMP version still holds: **clock-aligned boundaries (every 10 min) work better than break-triggered retros**. You can't skip them by staying in the flow.

The Claude Code version adds one thing the OMP version couldn't: the retro fires **from inside the agent's own turn**, not from an external process injecting text into the terminal.

---

## How it works

Three hooks wire into CC's lifecycle:

```
SessionStart  → retro-tracker.py start        # initialize segment timer
PostToolUse   → retro-tracker.py tick  (async) # count tool calls
PreToolUse    → retro-tracker.py check block   # gate: fire retro if boundary passed
Stop          → retro-tracker.py check (asyncRewake) # also check on session end
```

`retro-tracker.py` maintains a state file at `~/.claude/agent-retro-state.json`:

```json
{
  "segment_start": "2026-06-21T19:51:55",
  "tool_counts": {"Bash": 4, "Read": 2, "Glob": 1},
  "next_boundary_ms": 1782052200000,
  "last_retro": "2026-06-21T19:51:55"
}
```

At every `PreToolUse` event, the script checks: has `next_boundary_ms` passed? If yes, and if there was activity (tool calls > 0), it fires the retro.

---

## Two delivery paths

The tricky part is that there are two callers, and they need different behavior:

**`PreToolUse` (block mode):** The retro should block the tool call until Claude has run the retro. The script returns a `permissionDecision: "deny"` response with the retro prompt as the reason. CC shows this as a denied tool call with the full retro text. Claude runs the retro before attempting the tool again.

```python
print(json.dumps({
    "hookSpecificOutput": {
        "hookEventName": "PreToolUse",
        "permissionDecision": "deny",
        "permissionDecisionReason": message,
    }
}))
sys.exit(0)
```

**`Stop` hook (asyncRewake mode):** When the session ends, there's no tool call to block. Instead, the script exits with code 2 and prints the retro prompt to stdout. CC's `asyncRewake` feature interprets exit 2 + stdout as a new user turn — Claude wakes up and runs the retro before truly stopping.

```python
print(message, end="")
sys.exit(2)
```

Both paths reset the state immediately before emitting — so if both somehow fire near-simultaneously, only the first one across the boundary triggers a retro.

---

## The retro prompt

The script loads the retro text from a `SKILL.md` file in the repo (stripping YAML frontmatter). The injected message looks like:

```
Segment: 07:50:12 PM → 08:00:19 PM (10min), 23 calls (8 Bash, 7 Read, 5 Glob, 3 Edit)

You are running a Core Retro (10 min). Answer each question tightly — one paragraph max.

1. Done — What was accomplished? ...
2. Stuck? — Any backtracking or repeated failure? ...
3. Wheel — Did you check for an existing tool/script before writing code? ...
4. DRW — Anything to search for that would save tokens? ...
5. Knowledge gem — Web search one adjacent tool or technique ...
6. Agentic Hygiene — Should this continue in a new session?
```

The segment header (time range, duration, tool call breakdown) gives Claude context on what just happened — without requiring it to re-read files.

---

## The `rules/retro.md` enforcement layer

The hook fires the retro, but Claude needs to actually run it rather than summarize or skip it. A rule file at `~/.claude/rules/retro.md` makes this explicit:

```markdown
When a retro system-reminder arrives (text starting with "Segment: HH:MM →" or
containing "You are running a **Core Retro**"), you MUST:
1. Stop whatever you were doing
2. Run the retro visibly in your response — answer all 6 questions
3. Only then continue with the user's request

Retros are first priority. Do not silently absorb and ignore them.
```

Without this, Claude would sometimes acknowledge the retro in one sentence and continue working. The rule closes that gap.

---

## What didn't happen here

The earlier approach used `retro-injector.ps1` — a PowerShell script that ran as a background job, waited for 10-minute wall-clock boundaries, found the Claude pane via `wezterm cli list`, and injected `/retro` via `wezterm cli send-text`. It worked, but:

- Required WezTerm specifically
- Ran as a separate process that had to be started manually
- Could miss the pane if CC was in a different window
- Was a polling loop, not event-driven

The hooks version has none of these problems. It's event-driven, terminal-agnostic, and starts automatically with every CC session.

---

## Configuration

Interval is configurable via `retro.config.json` in the repo root:

```json
{"interval_ms": 600000}
```

600,000ms = 10 minutes. Change to 300,000 for 5-minute retros during focused sprints.

---

## Settings wiring

```json
"hooks": {
  "SessionStart": [{"type": "command", "command": "python .../retro-tracker.py start", "async": true}],
  "PreToolUse":   [{"type": "command", "command": "python .../retro-tracker.py check block", "timeout": 10}],
  "PostToolUse":  [{"type": "command", "command": "python .../retro-tracker.py tick", "async": true}],
  "Stop":         [{"type": "command", "command": "python .../retro-tracker.py check", "timeout": 10, "asyncRewake": true}]
}
```

`PreToolUse` timeout is 10s — enough for the file read and boundary check. `PostToolUse` is async because counting tool calls doesn't need to block.

---

## What a fired retro looks like

```
Claude Code — PreToolUse hook denied a Bash call:

Segment: 07:50 PM → 08:00 PM (10min), 23 calls (8 Bash, 7 Read, 5 Glob, 3 Edit)

You are running a Core Retro...

[Claude's retro response]

1. Done — Confirmed retro system undocumented in blog.md; gathered all
   implementation details. Worth it.
2. Stuck? — One wrong path: /c/Users/... instead of C:/Users/... for settings.json.
   Caught fast.
3. Wheel — No code written, only reading. Clean.
...

[Claude resumes the original tool call]
```

The denied tool call appears in the transcript. Claude runs the retro, then the original action continues. From the user's perspective: you see a retro appear mid-stream, Claude answers it, work resumes.

---

## Repo

`github.com/ankitg12/agent-retro-cc`

Structure:
```
scripts/retro-tracker.py   # the hook script
skills/retro/SKILL.md      # the retro prompt
retro.config.json          # interval config
```

---

## Related posts

- [Clock-aligned retrospection for AI coding agents](https://ankitg12.github.io/ai/agents/productivity/2026/05/06/clock-aligned-retro-for-ai-agents.html) — the OMP version, WezTerm injection
- [Break-time retrospection for AI coding agents](https://ankitg12.github.io/ai/agents/productivity/2026/04/27/break-time-retro-for-ai-agents.html) — the original concept
- [claude-code-timestamps](https://ankitg12.github.io/ai/agents/productivity/2026/05/29/claude-code-timestamps.html) — the hook that runs alongside this one

---

> Generated with [Claude](https://claude.ai) (Anthropic) based on a real session with @ankitg12.
> #claude #ai #llm #aiassisted #agenticai #claudecode
