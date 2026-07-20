---
layout: post
title: "Building a quiet /pause for OMP — and discovering how much of the feature set I didn't know"
date: 2026-07-20
categories: ai agents productivity tools
series: "AI coding agent productivity"
---

I wanted to pause my AI coding agent — freeze it mid-session so I could step away or read something on screen without it charging ahead — and resume later with `/resume` or `/unpause`. Simple ask. It took a wrong build, a scrapped extension, a real concurrency bug, and an upstream issue before landing on something worth keeping.

## First attempt: reinventing something that already existed

My first instinct was to write an extension from scratch: hook `before_agent_start` and `tool_call`, hold an unresolved promise until a `/resume` command released it. Textbook cooperative pause gate. I built it, wired it into config, and mid-smoke-test a full-screen "P A U S E D" splash appeared that I hadn't written.

OMP already ships a native `/pause`. It freezes the main agent, every subagent, *and* the advisor via a process-global gate (`agentPauseGate`, exported from `@oh-my-pi/pi-agent-core`), polled at the two safe boundaries every agent loop passes through: before a model call, and before a tool call starts. My hand-rolled version only guarded the current session's own two events — it didn't touch subagents or the advisor at all. Strictly worse, and redundant.

Lesson, not for the first time: grep the tool's own source for the feature name before building it.

```bash
node -e "
const fs=require('fs'), path=require('path');
function walk(dir){
  for (const f of fs.readdirSync(dir, {withFileTypes:true})) {
    const p = path.join(dir, f.name);
    if (f.isDirectory()) walk(p);
    else if (f.name.endsWith('.js') && fs.readFileSync(p,'utf8').includes('PAUSED')) console.log(p);
  }
}
walk('node_modules/@oh-my-pi/pi-coding-agent');
"
```

I deleted the extension, the config file, and the log. Total loss: about twenty minutes and some context.

## The actual gap: fullscreen only

Native `/pause` does exactly two things in its handler: `agentPauseGate.pause()`, then `await runPauseScreen(ctx)` — and `runPauseScreen` is a hardcoded fullscreen overlay (`fullscreen: true`, `anchor: "top-left"`, `maxHeight: "100%"`). No flag, no config key, no seam. If you want the freeze without losing sight of your terminal — say, you're mid-read of a long diff or log output and just want to stop the agent from touching anything for a minute — there's no way to ask for that from the outside.

That's a real gap, and it's not something I can fix by forking OMP's source. So two things, in order:

1. File the actual feature request upstream, with exact file/line references, so it doesn't get lost as "works for me, closing."
2. Build the workaround I actually needed in the meantime — reusing the *same* gate the native command drives, just with a different display.

## Reuse the primitive, replace the display

`agentPauseGate` is a plain exported singleton with a small surface:

```typescript
export class AgentPauseGate {
  get paused(): boolean;
  get pausedAt(): number | undefined;
  pause(): boolean;                          // false if already paused
  resume(): number | undefined;              // pause duration in ms, or undefined
  onChange(listener: (paused: boolean) => void): () => void;
  async waitUntilResumed(signal?: AbortSignal): Promise<void>;
}
export const agentPauseGate = new AgentPauseGate();
```

Since it's importable from `@oh-my-pi/pi-agent-core` — the same package the native command imports it from — an extension can drive it directly. Freeze semantics become identical to the builtin (main + subagents + advisor, parked at the next safe boundary) without touching a single line of OMP's own source. I named the commands `/hold` / `/unhold` / `/hold-status` rather than `/pause`/`/resume` — OMP's extension loader silently skips (with a warning) any extension command whose name collides with a builtin, so there's no "override" path; different names are the only option.

The display side is a `setWidget` call instead of the fullscreen component — a persistent, ticking status line above the editor instead of a takeover screen:

```typescript
function widgetLines(): string[] {
  const heldMs = agentPauseGate.pausedAt !== undefined
    ? Date.now() - agentPauseGate.pausedAt : 0;
  return [`⏸ held for ${formatClock(heldMs)} — /unhold to continue (transcript stays visible)`];
}

pi.registerCommand("hold", {
  description: "Pause the agent without covering the screen",
  handler: (_args, ctx) => {
    if (!agentPauseGate.pause()) {
      ctx.ui.notify("[hold] already held", "info");
      return;
    }
    startTicking(ctx); // ctx.setInterval(...) updating the widget every second
    ctx.ui.notify("[hold] agent held — use /unhold to continue", "info");
  },
});
```

## The bug that mattered: a race at the turn boundary

First live test looked right — the widget appeared, ticked, no fullscreen takeover. Then I typed a prompt while held: `say the word banana`. It answered anyway.

`agentPauseGate` is polled only *inside* `agent-loop.ts`, at its own internal checkpoints:

```typescript
// before each model call within a turn
if (agentPauseGate.paused) await agentPauseGate.waitUntilResumed(signal);
...
// before each tool call
if (agentPauseGate.paused) await agentPauseGate.waitUntilResumed(record.signal);
```

Sending `/hold` and then immediately submitting a plain-text prompt can race the loop's first checkpoint — if the loop already grabbed its "not paused" read before `pause()` landed, a text-only turn (no tool calls, so the second checkpoint never fires) sails straight through. The fix was a second, earlier gate — the extension's own `before_agent_start` hook, which fires before the agent is dispatched at all:

```typescript
pi.on("before_agent_start", async (_event, ctx) => {
  if (!agentPauseGate.paused) return;
  await agentPauseGate.waitUntilResumed();
});
```

Verified in a live session afterward: `/hold` → `say the word mango` → the "Working…" spinner sat there indefinitely, no output — until `/unhold`, at which point the queued prompt executed and "mango" printed. Belt and suspenders, but the belt was load-bearing.

## Finding the rest of the iceberg

Chasing down `/pause` meant reading OMP's entire builtin command registry, and it turned up a lot I'd never used in months of daily driving:

| Command | What it actually does |
|---|---|
| `/goal set <objective>` | Persistent autonomous objective for the session, with its own token budget (`/goal budget <N\|off>`), pause/resume/drop |
| `/branch`, `/fork`, `/tree` | Branch a new session off a *previous* message, not just "now" — git branching for conversations |
| `/shake [elide\|images]` | Strip heavy content (tool results, images) from context without a full `/compact` |
| `/collab [start\|view\|stop]` + `/join <link>` | Share a live session via a relay; others watch or actively join |
| `/btw <question>` | Ephemeral side question using current context, doesn't pollute the main transcript |
| `/tan <work>` | Spawn a full background agent on tangential work while the main thread keeps going |
| `/omfg <complaint>` | Forges a rule from a complaint to stop a recurring agent behavior |
| `/memory mm list/show/refresh/history` | Direct inspection of individual "mental models" in the memory bank |

None of these were hidden exactly — they're all in `--help` and `/hotkeys` — but I'd never gone looking. The `/pause` rabbit hole is what forced the read.

## Takeaways

- **Grep before you build.** A five-minute search of the tool's own source would have saved the first scrapped extension entirely.
- **A missing feature and a missing config knob are different bugs.** `/pause`'s freeze mechanism was exactly right; only its display was inflexible. Filing an upstream issue for the display, and building a stopgap against the *real* shared primitive, beat reimplementing the whole thing badly.
- **Cooperative gates race at boundaries you don't own.** If a shared pause/cancel primitive is polled only inside someone else's loop, don't assume your own trigger point is inside that loop too — verify with an actual concurrent test, not a single happy-path try.

## Source

- Extension: `agent-pause-omp` (personal, unpublished) — commands `/hold` / `/unhold` / `/hold-status`
- Upstream feature request: [oh-my-pi#6101](https://github.com/can1357/oh-my-pi/issues/6101) — decouple `agentPauseGate` engagement from the fullscreen `runPauseScreen` display
- Prior post in this series: {% post_url 2026-05-11-slowmo-for-ai-coding-agents %}
