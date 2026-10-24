---
layout: post
title: "Your agent forgets everything between sessions. I built it dreams."
date: 2026-10-24
categories: ai agents productivity
series: "AI coding agent productivity"
---

Every session with my coding agent starts from zero. Preferences I stated last week, tool quirks I diagnosed last month, corrections I made yesterday — all gone. The agent re-learns them by annoying me again.

Anthropic's answer is [Dreams](https://platform.claude.com/docs/en/managed-agents/dreams): async jobs that mine 1–100 past sessions for durable lessons and produce a memory store you review before swapping in. [OMP](https://github.com/can1357/oh-my-pi) has nothing like it. Its `/memory` pipeline consolidates incrementally, session by session, and the one attempt at idle-time consolidation — [PR #8716](https://github.com/can1357/oh-my-pi/pull/8716), open since August — hasn't merged. I didn't want to maintain a fork to get one feature. So the whole thing lives in user space: an extension, a skill file, two scripts, and a cron job. No fork, no upstream patch, survives `omp update`.

The extension is [agent-dream-omp](https://github.com/ankitg12/omp-extensions). First verified run, this morning: 10 sessions scanned, 14 memories written, one diary entry. The memories were real — config conventions I'd stated across different sessions, corrections I'd made to agent behavior, repo-layout rules I'd laid down weeks apart and never written anywhere. That last category is the one that stung. I had rules I was enforcing and nowhere to keep them.

The interesting part isn't the consolidation. It's the two ways the build failed before it worked, because both failures are about how OMP extensions actually behave versus how the docs say they behave.

The first failure was silent. My extension injected the consolidation prompt with `sendUserMessage(text, { deliverAs: "followUp" })` — the documented way to queue a message. Six passes injected across two sessions. Zero executed. The SDK types contain the answer in one sentence, buried in the `ExtensionContext` docs: *"When deliverAs is set, queue the message instead of starting a new turn."* Queued is not run. My six passes sat in the queue waiting for a user turn that never referenced them, and the extension logged each one as a success because delivery had succeeded. The state file said `totalPasses: 6, lastSessions: null` — injected six times, executed zero. That null was the whole story.

The fix is `sendMessage({ customType, content, display: true }, { triggerTurn: true, deliverAs: "nextTurn" })`. `triggerTurn: true` "schedules an internal continuation" — as far as I can tell, the only extension API that starts model work rather than queuing text near it. No bundled extension uses it. The model treats the `role: "custom"` message as an instruction and goes to work with file tools. That gap between the docs and the shipped SDK is now [oh-my-pi#16924](https://github.com/can1357/oh-my-pi/issues/16924): the documented managed timers (`ctx.setTimeout`/`ctx.clearTimer`) don't exist in `pi-coding-agent@16.1.16` either, so every bundled extension that needs a timer — `agent-afk`, `session-reminder` — ships the raw `setInterval` pattern the docs warn will take down the session.

The second failure was cost. Once the pass actually ran, I watched it burn a dozen tool calls in two minutes — four or five just listing session files and sorting by mtime across two directory levels, then more calls discovering the memory backend config, paths it could have been told. It did good extraction work and then kept digging past the point of diminishing returns, because nothing told it to stop. Three changes fixed the economics, and they're all about moving knowledge from discovery to injection:

A `dream-index` script prints `mtime lines path` newest-first in one call — the entire discovery phase collapses into it. A `dream-diary` script takes an entry body and handles prepend-plus-trim atomically, so the model never does read-modify-write surgery on a growing file. And the injected prompt carries a tool budget ("~15 calls; report partial rather than grinding"), a stop rule (quit after two consecutive sessions with zero new signal), and the exact memory target resolved by the extension from config — including the option of no memory file at all, in which case lessons go to the diary and the extension is forbidden from inventing paths.

That last point generalizes. The write path is not the product; the retrieval path is. Testing that the diary got an entry proves filing works. It says nothing about whether the next session is smarter. The pattern that closed the loop came from [Chanl's writeup of the same problem](https://www.channel.tel/blog/agent-async-memory-consolidation-dreaming): gate the write, test the read. The skill carries an extraction blocklist — never security exceptions, access grants, auth bypasses, credentials, one-time secrets, transient states, anything uncertain, anything expiring under a week, with "when in doubt, discard" as the default — because a transcript containing "skip verification for this account" must not become a memory. And the skill ends by writing a report JSON the extension folds into state on next start, so `/dream status` reports what actually happened instead of what was injected. The documented verification is a four-session loop: plant a preference, consolidate, check a fresh session uses it, reverse the preference, consolidate, check the fresh session follows the new value.

One honest boundary: the idle-timer auto-fire — the thing that makes this "dreams" rather than "a command I run" — hasn't been verified end to end. Manual `/dream` works, the cron script works, the timer arms and the gate logic is tested. Whether a 30-minute-idle session actually wakes up and consolidates unattended is still open. I'm running it, not claiming it.

The code, the skill, and both scripts are in the public repo. If the idle path verifies, the post gets an update. If it doesn't, the post gets a more interesting update.

---

## How it is wired

Three triggers, one pass:

| Trigger | Mechanism |
|---|---|
| Idle 30 min | in-session timer, re-armed on every turn |
| `/dream` | slash command (`now` / `status` / `diary` / `log`) |
| cron | `omp-dream` headless script, no live session needed |

Config lives in `~/.omp/agent/agent-dream.json` (all fields optional): `idleMs`, `minIntervalMs`, `diaryPath`, `maxSessions`, `maxDiaryEntries`, `maxMemoriesPerPass`, `maxDiaryEntryLines`, `logPath`, `memoryFile` (empty = diary-only), `reportPath`, `enabled`, `debug`. Last-pass state in `agent-dream-state.json`; the skill's report JSON folds into it on next start.

---

## Files and references

- Extension: [github.com/ankitg12/omp-extensions](https://github.com/ankitg12/omp-extensions) (`packages/agent-dream-omp/`) — `dream.ts`, `dream-state.ts`, `SKILL.md`, `dream-index`, `dream-diary`, `omp-dream`
- Upstream timer-API gap: [oh-my-pi#16924](https://github.com/can1357/oh-my-pi/issues/16924)
- Unmerged upstream PR, same idea, pipeline-integrated: [oh-my-pi#8716](https://github.com/can1357/oh-my-pi/pull/8716)
- Retrieval-path pattern: [Chanl on async consolidation](https://www.channel.tel/blog/agent-async-memory-consolidation-dreaming)
- The API it mirrors: [Anthropic Dreams](https://platform.claude.com/docs/en/managed-agents/dreams)
