---
layout: post
title: "Session review for AI coding agents"
date: 2026-05-12
categories: ai agents productivity
series: "AI coding agent productivity"
---

The session ends. The context evaporates. You remember you made progress. You don't remember which files you touched, which approach you discarded, or what you decided to do tomorrow.

The agent doesn't remember either. The next session starts cold.

This post is about recovering that memory from the log files OMP already writes — structured, timestamped, searchable — and turning them into a digest before you've forgotten what happened.

See also: [Per-session goal tracking]({% post_url 2026-05-11-session-goal-tracking-for-ai-agents %}), [Clock-aligned retrospection]({% post_url 2026-05-06-clock-aligned-retro-for-ai-agents %}), [Personal Agentic Process]({% post_url 2026-05-06-personal-agentic-process %}).

---

## The problem

Every OMP session is a `.jsonl` file. Every user message, tool call, and assistant response is a timestamped JSON line. The files accumulate in `~/.omp/agent/sessions/`. Nobody reads them.

The failure mode: you spent eight hours on a problem, left three open threads, made a consequential design decision, and stepped away for the night. The next morning you stare at your editor and try to reconstruct the state from memory. You miss something. It costs you an hour in the next session to re-derive it.

The fix is not better memory. It's extraction.

---

## What OMP stores

Session files land at:

```
~/.omp/agent/sessions/<cwd-slug>/<timestamp>_<uuid>.jsonl
```

Each line is a JSON object: `role` (`user` | `assistant`), `content`, and an embedded wall-clock timestamp OMP injects into every user message:

```
[May 11 14:41:00 PM] First, check if today's daily goal is set:
```

The filename encodes the session start time:

```
2026-05-11T09-08-35-411Z_019e164b-9fd0-7000-be52-fd6d1d2628a9.jsonl
```

Subagent sessions nest under a deeper path (two or more `/` levels below the root session dir). Everything is structured enough to mine.

---

## The extractor

`extract.py` reads the session store, filters by time window, and emits per-session blocks: title, project, timestamps, duration, and all user messages that fall within the window — stripped of OMP's timestamp prefix and noise (retro injections, HTML comments, session-goal boilerplate).

```bash
python3 ~/.claude/skills/review-sessions/scripts/extract.py --since "yesterday"
```

Outputs:

```
SINCE: 2026-05-11 00:00 IST  until  2026-05-11 23:59 IST
SESSIONS: 7

=== SESSION 1/7 ===
Title   : OMP Session Variables and Persistence
Project : tmp
Time    : 2026-05-11 10:48 - 13:16  (2h 27m)
...
```

Time expressions are delegated entirely to `dateparser` — no custom parsing logic:

| Expression | Window |
|---|---|
| `yesterday` | yesterday 00:00 → 23:59 (local) |
| `today` | today 00:00 → now |
| `6 hours ago` | 6h ago → now |
| `2026-05-10` | that day 00:00 → 23:59 |

Filter to one project with `--project <name>`. Default limit is 30 sessions.

### What makes it OMP-specific

Three things in the extractor are OMP-specific and won't work against raw `claude` CLI sessions:

1. **Filename timestamp** — the regex `(\d{4})-(\d{2})-(\d{2})T(\d{2})-(\d{2})-(\d{2})-\d+Z_` matches OMP's naming scheme. Claude Code's project history uses a different layout.

2. **Timestamp prefix stripping** — OMP prepends `[May 11 14:41:00 PM]` to every user message. The extractor strips this before displaying. Claude Code's JSONL has no such prefix.

3. **Subagent detection** — OMP nests subagent sessions deeper in the directory tree. The extractor counts path depth to skip them when summarizing, so you get the top-level human turns, not the agent's internal delegation.

---

## The review skill

The `/review-sessions` skill wraps extraction with summarization. It runs the extractor, reads the output, and asks the agent to produce:

- **Per-session summary** — what was worked on, outcome or status, notable decision or blocker
- **Open threads** — anything left unresolved or explicitly deferred
- **Action items** — concrete next steps derived from session content

```
/review-sessions           # yesterday (default)
/review-sessions today     # end-of-day digest
/review-sessions last 6h   # rolling review
/review-sessions 2026-05-10
```

The output is a dense digest, not a transcript. Seven sessions from yesterday collapsed to one read in under a minute.

---

## What this unlocks

**Morning standup with yourself.** Before opening any editor, run `/review-sessions yesterday`. You know exactly what's done, what's open, and what the first move is. No reconstruction.

**Handoff continuity.** If you switch contexts mid-day — from infra work to code review — run `/review-sessions last 6h` before the switch. The open threads are in the digest, not in your head.

**Pattern spotting.** Over time the digests reveal structural patterns: which kinds of sessions end in open threads, which times of day produce the most drift, which project areas keep generating the same blockers. The raw material is already there.

---

## What it doesn't do

The extractor doesn't read assistant turns — only user messages. That means tool call results, agent analysis, and intermediate reasoning are invisible. If a critical insight lives only in an assistant turn (rare, but possible), the digest won't surface it. The workaround: write decisions into the session goal or a notes file at the time they're made.

It also doesn't cross-reference session goals. A future version that reads the `.goal.md` files alongside the JSONL could add a verdict per session — goal achieved, drifted, or abandoned — for a richer retrospective. The data is already there.

---

## Requirements

```bash
pip install dateparser
```

Python 3 in PATH. The skill reads `~/.omp/agent/sessions/**/*.jsonl` — no other setup.

---

## Source

- [claude-config](https://github.com/ankitg12/claude-config) — `skills/review-sessions/` including `extract.py`
- [agentpatterns.ai: Session Recap](https://agentpatterns.ai/agent-design/session-recap/) — goal-shaped handoff at context boundaries
- [dateparser](https://dateparser.readthedocs.io/) — natural language time parsing used by the extractor
