---
layout: post
title: "Adding timestamps to AI coding agent conversations"
date: 2026-04-27
categories: ai agents productivity
---

AI coding agents don't show when things happened. You scroll back through a long session and can't tell if that tool call was 2 minutes ago or 20. The agent itself has no concept of elapsed time — it can't notice it's been stuck for 15 minutes.

This is the [most requested feature](https://github.com/anthropics/claude-code/issues/44763) on Claude Code — 10+ duplicate issues, none implemented. Here's a 40-line [OMP](https://github.com/can1357/oh-my-pi) extension that does it.

---

## What it does

- **User messages**: `[04:42:41 PM] your message here` — timestamp prepended deterministically
- **Agent responses**: system prompt injects current time, instructs agent to prepend `[HH:MM:SS]`
- **Status bar**: live `[04:42:45 PM] bash` during tool execution, `[04:43:12 PM] 27s` after turn

## How it works

Three OMP extension hooks:

**1. Input transform** — deterministic, modifies user text before it reaches the LLM:

```typescript
pi.on("input", (event) => {
  return { action: "transform", text: `[${ts()}] ${event.text}` };
});
```

**2. System prompt injection** — tells the agent the current time:

```typescript
pi.on("before_agent_start", (event) => {
  return {
    systemPrompt: event.systemPrompt +
      `\n\nCurrent time: ${new Date().toLocaleTimeString()}. ` +
      `Always prepend [HH:MM:SS] timestamp to your responses.`
  };
});
```

This fires before each agent loop. OMP rebuilds the system prompt fresh each time — no accumulation.

**3. Status bar** — elapsed time per turn:

```typescript
pi.on("turn_start", (_event, ctx) => {
  turnStartMs = Date.now();
  ctx.ui.setStatus("timestamp", `[${ts()}]`);
});

pi.on("turn_end", (_event, ctx) => {
  const elapsed = Math.round((Date.now() - turnStartMs) / 1000);
  ctx.ui.setStatus("timestamp", `[${ts()}] ${elapsed}s`);
});
```

## What didn't work

**`pi.sendMessage()` from event handlers.** The plan was to inject a timestamp entry after each turn — deterministic, no instruction following. But `sendMessage` and `appendEntry` silently fail from event handlers. [Known OMP issue](https://github.com/badlogic/pi-mono/issues/2860). The `input` event transform is the only reliable injection mechanism.

**`/reload` for extension development.** OMP caches compiled TypeScript modules. Code changes require a full process restart, not just `/reload`.

## Setup

```bash
# Add to ~/.omp/agent/config.yml
extensions:
  - ~/tools/chat-timestamps
```

The full extension is [~40 lines of TypeScript](https://github.com/ankitg12/chat-timestamps).

---

*The agent having time awareness changes its behavior. It can notice "this segment took 39 seconds" vs "this took 8 minutes." It correlates effort with progress. That's the reasoning primitive [issue #32566](https://github.com/anthropics/claude-code/issues/32566) was asking for.*
