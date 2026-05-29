---
layout: post
title: "Claude Code timestamps: 12 lifecycle events, one script"
date: 2026-05-29
categories: ai agents productivity
series: "AI coding agent productivity"
---

[Earlier this year]({% post_url 2026-04-27-timestamps-for-ai-coding-agents %}) I added timestamps to AI coding agent sessions using an OMP extension — three hooks, ~40 lines of TypeScript, injecting wall-clock time into the status bar and system prompt. It worked, but it was Pi-specific. Claude Code has its own native hook system that goes deeper: 12 lifecycle events, each with a JSON contract, all reachable without touching OMP at all.

This post covers the CC-native version: what events exist, which ones are worth stamping, what the script does with them, and two design decisions that almost went wrong.

---

## What CC hooks give you

Claude Code fires hooks at specific points in its execution cycle. Each hook receives JSON on stdin and can write JSON to stdout — `systemMessage` to inject into context, `additionalContext` to append to tool output, `permissionDecision` to block or allow.

The full lifecycle:

| Event | When |
|---|---|
| `SessionStart` | Session begins or resumes |
| `UserPromptSubmit` | User sends a message |
| `PreToolUse` | Before each tool call |
| `PostToolUse` | After each tool call succeeds |
| `PostToolUseFailure` | After a tool call fails |
| `PostToolBatch` | After a parallel batch of tools completes |
| `SubagentStart` | Subagent spawned |
| `SubagentStop` | Subagent finishes |
| `TaskCreated` | Task created |
| `TaskCompleted` | Task completed |
| `PreCompact` | Before context compaction |
| `PostCompact` | After context compaction |
| `Stop` | Response complete |
| `SessionEnd` | Session closes |

The OMP timestamps extension covered three moments: input, system prompt injection, and status bar. CC hooks cover fourteen.

---

## Which events are worth stamping

Not all fourteen. Two traps:

**`PreToolUse` + `PostToolUse` + `PostToolBatch`** = three timestamps per tool call. `PreToolUse` fires before every tool. `PostToolBatch` fires after every batch — including batches of one. Sequential tool calls produce `→ read starting / read completed / batch resolved` for every single tool. Three lines where one is enough. Drop `PreToolUse` and `PostToolBatch`.

**`StopFailure`** explicitly says "output and exit code are ignored" in the docs. Skip it.

That leaves 12 events that produce visible, meaningful output. The script handles all of them.

---

## The script

`timestamps.py` — one file, stdlib only, no pip install:

```python
# Stop: turn elapsed + running session total
elif mode == "stop":
    data = read_stdin()
    sid = data.get("session_id", "default")
    turn_f, _, total_f = timing_files(sid)
    turn_part = ""
    session_part = ""
    try:
        if turn_f.exists():
            elapsed = t - float(turn_f.read_text(encoding="utf-8"))
            turn_part = f" ({fmt_secs(elapsed)})"
            total = float(total_f.read_text(encoding="utf-8")) if total_f.exists() else 0.0
            total += elapsed
            total_f.write_text(str(total), encoding="utf-8")
            turn_f.unlink(missing_ok=True)
            session_part = f" | session {fmt_secs(total)}"
    except Exception:
        pass
    sys_msg(f"[{now()}] — response complete{turn_part}{session_part}")
```

What you see at the end of each turn:

```
[09:21:17 AM] — user message received (after 47m33s idle)
[09:21:19 AM] read completed
[09:21:44 AM] — response complete (27.3s) | session 4m12s
[09:22:01 AM] — session ending (logout) — Claude 4m12s | wall 47m33s
```

Four distinct moments, one number per line.

### Session state is keyed by session_id

State lives in `~/.claude/session_timing/<session_id>.{turn_start,session_start,total}`. Each CC session gets its own files. Two open windows don't clobber each other — a bug the global-file approach would have silently introduced.

### fmt_secs covers the full range

```python
def fmt_secs(secs: float) -> str:
    if secs < 1:
        return f"{secs * 1000:.0f}ms"
    if secs < 60:
        return f"{secs:.2f}s"
    mins, s = divmod(secs, 60)
    if mins < 60:
        return f"{int(mins)}m{s:.0f}s"
    h, m = divmod(mins, 60)
    return f"{int(h)}h{int(m)}m{s:.0f}s"
```

`250ms`, `3.14s`, `2m5s`, `1h3m5s` — covers a tool call taking 200ms and a session running three hours.

---

## Claude compute vs wall time

`SessionEnd` outputs two numbers:

```
[09:22:01 AM] — session ending (logout) — Claude 4m12s | wall 47m33s
```

**Claude time**: sum of all turn durations. How long the model was actually computing.  
**Wall time**: session open to session close. How long you had CC open.

The gap is your think/type/read time. A 47-minute session where Claude computed for 4 minutes means 43 minutes of human time — reviewing output, deciding next steps, getting coffee. That ratio matters for understanding where sessions actually go.

---

## Install

**Plugin (one command):**

```
/plugin add --from https://github.com/ankitg12/claude-code-timestamps
```

No settings.json editing — the plugin wires all 12 hooks automatically. Requires Python 3 on PATH as `python`.

**Manual — add to `~/.claude/settings.json`:**

```json
{
  "hooks": {
    "SessionStart":       [{"matcher":"","hooks":[{"type":"command","command":"python \"~/.claude/hooks/timestamps.py\" session_start","async":true}]}],
    "UserPromptSubmit":   [{"matcher":"","hooks":[{"type":"command","command":"python \"~/.claude/hooks/timestamps.py\" prompt"}]}],
    "PostToolUse":        [{"matcher":"","hooks":[{"type":"command","command":"python \"~/.claude/hooks/timestamps.py\" tool","async":true}]}],
    "PostToolUseFailure": [{"matcher":"","hooks":[{"type":"command","command":"python \"~/.claude/hooks/timestamps.py\" tool_fail"}]}],
    "SubagentStart":      [{"matcher":"","hooks":[{"type":"command","command":"python \"~/.claude/hooks/timestamps.py\" subagent_start","async":true}]}],
    "SubagentStop":       [{"matcher":"","hooks":[{"type":"command","command":"python \"~/.claude/hooks/timestamps.py\" subagent_stop"}]}],
    "TaskCreated":        [{"matcher":"","hooks":[{"type":"command","command":"python \"~/.claude/hooks/timestamps.py\" task_create","async":true}]}],
    "TaskCompleted":      [{"matcher":"","hooks":[{"type":"command","command":"python \"~/.claude/hooks/timestamps.py\" task_done"}]}],
    "PreCompact":         [{"matcher":"","hooks":[{"type":"command","command":"python \"~/.claude/hooks/timestamps.py\" pre_compact"}]}],
    "PostCompact":        [{"matcher":"","hooks":[{"type":"command","command":"python \"~/.claude/hooks/timestamps.py\" post_compact"}]}],
    "Stop":               [{"matcher":"","hooks":[{"type":"command","command":"python \"~/.claude/hooks/timestamps.py\" stop"}]}],
    "SessionEnd":         [{"matcher":"","hooks":[{"type":"command","command":"python \"~/.claude/hooks/timestamps.py\" session_end","async":true}]}]
  }
}
```

> **Windows:** replace `~/.claude/hooks/timestamps.py` with the full path. Run `/hooks` to reload — no restart.

---

## What didn't work

**One entry point for all events.** CC doesn't include an `event_name` field in the common hook input. You can partially distinguish events by which fields are present (`tool_name` → tool event, `agent_type` → subagent event), but `SessionStart` and `PreCompact` both carry a `trigger` field with different semantics. The heuristic breaks at the edges. Explicit mode arg is cleaner.

**`PostToolBatch` for parallel efficiency measurement.** The idea was to timestamp when a parallel tool batch resolved — useful for measuring how long 5 concurrent reads took vs sequential. The problem: `PostToolBatch` fires for batches of one too. Every single sequential tool call produces an extra "batch resolved" stamp on top of "tool completed." Dropped.

---

## Prior art

[Issue #44763](https://github.com/anthropics/claude-code/issues/44763) — "Show timestamps on conversation messages" — is the active upstream thread, open since April 2026, 15+ comments, not yet shipped. Several community tools have emerged from it:

**[s-a-s-k-i-a/claude-code-timestamps](https://github.com/s-a-s-k-i-a/claude-code-timestamps)** — a `/timestamps` command that reads your session's JSONL transcript and renders a retrospective message timeline: `14:02 You / 14:05 Claude`. Complementary to this script — it shows *history*, while `timestamps.py` stamps *in real time*. Worth running both.

**[clankercode/claude-inject-idle-time](https://github.com/clankercode/claude-inject-idle-time)** — a plugin with a live statusline elapsed timer. The idle-gap annotation pattern is now also in `timestamps.py` (learned from this project). One thing `timestamps.py` still doesn't do: statusline elapsed timer.

**[pleasedodisturb fork](https://github.com/pleasedodisturb/claude-inject-idle-time/tree/add-mcp-time-tools)** — combines both of the above plus MCP time-query tools and the `/timestamps N` retrospective view in one install: `claude plugin add --from https://github.com/pleasedodisturb/claude-inject-idle-time`.

**[claude-code-session-timer-hook](https://github.com/jeecgboot/claude-code-session-timer-hook)** — 4-event session timer (SessionStart, UserPromptSubmit, Stop, SessionEnd), outputs `Turn: 4.72s · Total 1m18s`. The design `timestamps.py` started from, extended to 12 events with the session timing consolidated into a single Stop line.

**[claude-hook-utils](https://github.com/RasmusGodske/claude-hook-utils)** — typed Python dataclasses for hook stdin; eliminates `data.get(...)` in complex validators. Covers 4 events, requires pip install. Not used here — the `data.get()` calls in `timestamps.py` are trivially simple and adding a dependency would break the zero-dep install.

**[cc-safe-setup](https://github.com/yurukusa/cc-safe-setup)** — 677 Shell hooks covering safety and autonomy: destructive command blocking, secret scanning, context monitoring. No timestamp hooks; orthogonal.

---

## Source

- [claude-code-timestamps](https://github.com/ankitg12/claude-code-timestamps) — the script, English + Hindi README, copy-paste install block
- [Timestamps for AI coding agents]({% post_url 2026-04-27-timestamps-for-ai-coding-agents %}) — the OMP/Pi version this builds on (TypeScript, status bar, system prompt injection)
- [s-a-s-k-i-a/claude-code-timestamps](https://github.com/s-a-s-k-i-a/claude-code-timestamps) — retrospective transcript timeline; pairs well with this script
- [clankercode/claude-inject-idle-time](https://github.com/clankercode/claude-inject-idle-time) — statusline elapsed timer and idle-gap injection; covers what this script doesn't
- [Clock-aligned retrospection]({% post_url 2026-05-06-clock-aligned-retro-for-ai-agents %}) — why session timing matters: structured interruption at known intervals
- [Personal Agentic Process]({% post_url 2026-05-06-personal-agentic-process %}) — the broader discipline timestamps feed into
- [Claude Code hooks reference](https://code.claude.com/docs/en/hooks) — full event schema and output format documentation
- [GitHub issue #44763](https://github.com/anthropics/claude-code/issues/44763) — the upstream request; community workarounds documented in comments
