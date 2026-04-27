---
layout: post
title: "Break-time retrospection for AI coding agents"
date: 2026-04-27
categories: ai agents productivity
---

AI coding agents run for hours without pausing to reflect. Humans take breaks. This post describes wiring the two together: when [Stretchly](https://hovancik.net/stretchly/) fires a break, the agent stops, reviews what it did, and answers five standup questions before continuing.

Built as two [OMP](https://github.com/can1357/oh-my-pi) extensions. No external dependencies.

---

## The problem

Long coding sessions with AI agents have no natural checkpoints. The agent keeps going — sometimes in circles — without stepping back. You come back from a break and have no idea what happened, what worked, or what was wasted effort.

## The design

**stretchly-sync** already pauses tool execution during Stretchly breaks. The addition: snapshot the activity buffer when a break fires, then inject a configurable prompt as a follow-up message after the break ends.

```
tool call → tool call → tool call → [BREAK] → snapshot → wait → resume → retro prompt → continue
```

The retro prompt is configurable via `stretchly-sync.json`:

```json
{
  "onBreak": "~/.claude/commands/retro.md"
}
```

Point it at any file — a retro template, a checklist, a standup format. The extension reads it, strips YAML frontmatter, prepends an activity summary, and sends it as a follow-up user message via `pi.sendUserMessage()`.

## The retro prompt

`~/.claude/commands/retro.md`:

```markdown
---
description: Review work done since last break
---
You are running a break-time retrospection. Answer these questions concisely:

1. **Done**: What was accomplished in this segment?
2. **Efficiency**: What could have been done faster or more directly?
3. **Stuck points**: Any repeated tools, failures, or backtracking?
4. **Web search**: Would a search unblock the current task? If yes, what?
5. **What we can do better next**

and do it
```

Also works as `/retro` from the command palette for manual invocation.

## Activity tracking

Every `tool_call` event logs `{ timestamp, toolName }` into an in-memory buffer. On break, the buffer is snapshotted to `~/.omp/agent/retro-pending.json` and reset. The retro prompt gets a one-line summary:

```
Segment: 8min, 19 tool calls (11 bash, 3 write, 2 web_search, 2 read, 1 grep)
```

## Break detection

Breaks are detected at multiple points to avoid missing them:

- `tool_call` — before each tool execution
- `context` — before each LLM call
- `turn_start` / `turn_end` — at turn boundaries
- Idle poll (`setInterval`) — catches breaks when no events fire

A synchronous `checking` flag prevents the same break from triggering multiple retros.

## Timestamps extension

A second extension (`chat-timestamps`) adds timestamps to the chat:

- **User messages**: `[HH:MM:SS]` prepended via `input` event transform (deterministic)
- **Agent responses**: system prompt instruction via `before_agent_start` (best effort)
- **Status bar**: live `[HH:MM:SS] toolName` during execution, `[HH:MM:SS] Xs` after turn

```typescript
pi.on("input", (event) => {
  return { action: "transform", text: `[${ts()}] ${event.text}` };
});
```

This was the [most requested feature](https://github.com/anthropics/claude-code/issues/44763) on Claude Code with 10+ duplicate issues — and no existing implementation anywhere.

## What we learned building it

1. **OMP caches compiled extensions.** `/reload` doesn't bust the module cache. Full process restart required after code changes.

2. **`pi.sendMessage()` silently fails from event handlers.** Known issue ([#2860](https://github.com/badlogic/pi-mono/issues/2860)). `pi.sendUserMessage()` with `deliverAs: "followUp"` works. `input` event transform is the only reliable injection point.

3. **Always compile-check before committing.** A missing `}` silently prevented the extension from loading for 5 reload cycles. `esbuild --bundle` catches this instantly.

4. **Race conditions with multiple hooks.** Four event hooks + an idle poll all detecting the same break window = triple-firing. Fixed with a synchronous guard flag.

## Setup

```bash
# Clone
git clone https://github.com/ankitg12/stretchly-sync ~/tools/stretchly-sync

# Config (~/.omp/agent/config.yml)
extensions:
  - ~/tools/stretchly-sync
  - ~/tools/chat-timestamps

# Config (~/.omp/agent/stretchly-sync.json)
{
  "debug": true,
  "onBreak": "~/.claude/commands/retro.md"
}
```

Requires [Stretchly](https://hovancik.net/stretchly/) running and the [fixed-time breaks]({% post_url 2026-04-14-stretchly-fixed-time-breaks %}) scheduled task for clock-aligned breaks.

## Source

- [stretchly-sync](https://github.com/ankitg12/stretchly-sync) — break detection, activity tracking, configurable onBreak prompt
- [chat-timestamps](https://github.com/ankitg12/chat-timestamps) — per-message timestamps for OMP

---

*This post was written during a session where the retro extension was being built and tested — the retro fired multiple times during writing, catching its own bugs.*
