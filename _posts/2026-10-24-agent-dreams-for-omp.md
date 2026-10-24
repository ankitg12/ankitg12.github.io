---
layout: post
title: "Your agent forgets everything between sessions. I built it dreams."
date: 2026-10-24
categories: ai agents productivity
series: "AI coding agent productivity"
---

Every session with my coding agent starts from zero. Preferences I stated last week, tool quirks I diagnosed last month, corrections I made yesterday — all gone. The agent re-learns them by annoying me again.

Anthropic's answer is [Dreams](https://platform.claude.com/docs/en/managed-agents/dreams): async jobs that mine 1–100 past sessions for durable lessons and produce a memory store you review before swapping in. [OMP](https://github.com/can1357/oh-my-pi) has no equivalent — its `/memory` pipeline consolidates incrementally, and there's an [open PR (#8716)](https://github.com/can1357/oh-my-pi/pull/8716) for idle-time consolidation that hasn't merged. So I built it as a user-space extension. No fork.

---

## What it does

`agent-dream-omp` ([repo](https://github.com/ankitg12/omp-extensions)) runs a consolidation pass over recent session JSONLs: extract durable signal, update memory, prepend a `DREAMS.md` diary entry. Three triggers:

| Trigger | Mechanism |
|---|---|
| Idle 30 min | in-session timer, re-armed on every turn |
| `/dream` | slash command, manual |
| cron | `omp-dream` headless script, no live session |

First verified pass: 10 sessions scanned, 14 memories written, one diary entry. Real content — config conventions, user corrections, repo-layout rules mined from actual history.

---

## The one SDK discovery that shaped everything

My first build injected the pass with `sendUserMessage(text, { deliverAs: "followUp" })`. Six passes injected, zero executed. The SDK types say why, if you read them: *"When deliverAs is set, queue the message instead of starting a new turn."* Queued ≠ run. The passes sat in the queue waiting for a user turn that never referenced them.

The fix: `sendMessage({ customType, content, display: true }, { triggerTurn: true, deliverAs: "nextTurn" })`. `triggerTurn: true` "schedules an internal continuation" — the only extension API that starts model work. I filed [oh-my-pi#16924](https://github.com/can1357/oh-my-pi/issues/16924) separately: the docs describe managed `ctx.setTimeout`/`ctx.clearTimer` timers that don't exist in the shipped SDK, so every bundled extension uses raw timers the docs warn against.

---

## What the pass actually costs (and the scripts that cut it)

Watching the first working pass, the model burned ~12 tool calls in 2 minutes — 4–5 just listing and mtime-sorting session files across two directory levels, plus repeated backend-config discovery. Two `bun` scripts ship with the extension and the injected prompt orders their use:

- **`dream-index [--max N] [--exclude <id>]`** — session JSONLs newest-first (`mtime lines path`). One call replaces the whole discovery phase.
- **`dream-diary --diary <path> --max <n> --entry-file <tmp>`** — prepend entry + trim, atomically. No read-modify-write roulette on a growing file.

Plus two prompt rules that matter more than they sound: a ~15 tool-call budget ("report partial rather than grinding") and stop-after-two-zero-signal-sessions. The prompt also names the exact memory target — the extension resolves it from config, so the pass spends zero calls discovering backends. A `memoryFile` config field (empty = diary-only) means the extension never invents paths.

## The write path is not the product

Testing the write (cooldowns, caps, diary trim) is necessary and insufficient. The pattern that closed the loop came from [Chanl's writeup of the same problem](https://www.channel.tel/blog/agent-async-memory-consolidation-dreaming): gate the write path and test the retrieval path.

- **Extraction allowlist/blocklist in the skill.** Extract only: explicit preferences, stable facts, confirmed tool quirks, stated decision patterns, user corrections — each with a source. Never: security exceptions, access grants, auth bypasses, credentials, one-time secrets, transient states, uncertain facts, anything expiring under 7 days. *"When in doubt, discard — noisy retrieval is worse than no retrieval."* A session transcript containing "skip verification for this account" must not become a memory.
- **Report-back contract.** The skill writes `{"sessions", "memories", "diaryEntry", "at"}` JSON; the extension folds it into state on next start. `/dream status` reads state — truthful, not "last injected."
- **Retrieval test (documented, manual for now):** plant a preference in session one → `/dream now` → fresh session two must use it → session three states the opposite → `/dream now` → session four follows the new value.

## What's still unproven

The idle-timer auto-fire path hasn't been verified end-to-end — only manual `/dream` and the cron script. Framed honestly: manual + cron verified, idle-timer in testing. The consolidation it did is real either way.

---

## Files

- Extension: [github.com/ankitg12/omp-extensions](https://github.com/ankitg12/omp-extensions) (`packages/agent-dream-omp/`) — `dream.ts`, `dream-state.ts`, `SKILL.md`, `dream-index`, `dream-diary`, `omp-dream`
- Upstream issue (timer API gap): [oh-my-pi#16924](https://github.com/can1357/oh-my-pi/issues/16924)
- Unmerged upstream PR (same idea, pipeline-integrated): [oh-my-pi#8716](https://github.com/can1357/oh-my-pi/pull/8716)
