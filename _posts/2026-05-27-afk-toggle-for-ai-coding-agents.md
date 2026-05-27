---
layout: post
title: "AFK toggle for AI coding agents"
date: 2026-05-27
categories: ai agents productivity
series: "AI coding agent productivity"
---

The naive way to go AFK with an AI coding agent is to type your instructions and walk away. The problem is coming back: there's no signal to the agent that you've returned, no status indicator that autonomous mode is active, and no clean way to disengage. This is a 50-line OMP extension that gives you `/afk` to engage and `/back` to return.

---

## The problem

OMP slash commands backed by markdown files are prompt templates — they inject text into the conversation and that's it. The agent acts on the text, but the harness has no record of what happened. When you type `/afk`:

- No state is recorded anywhere
- The status bar shows nothing
- Typing `/afk` again sends the same prompt again instead of toggling
- When you return, you have to manually type "I'm back, stop and wait for me"

That's fine when you're at the keyboard. It's a minor annoyance when you actually step away.

---

## What it does

`/afk` engages autonomous mode:

1. Injects the AFK rules prompt into the conversation as a follow-up message
2. Sets a persistent `AFK 🔴` indicator in the status bar
3. Guards against double-engagement (second `/afk` notifies instead of re-injecting)

`/back` disengages:

1. Injects a "user is back, stop and wait" message
2. Clears the status indicator
3. Guards against calling it when not active

The status bar indicator is the key addition over the markdown approach — you can see at a glance whether AFK mode is on, even after returning to a session you left open.

---

## How it works

The extension registers two slash commands. Both use `pi.sendUserMessage` with `deliverAs: "followUp"` to inject a message into the conversation immediately after the handler returns:

```typescript
export default function agentAfk(pi: ExtensionAPI) {
  pi.setLabel("Agent AFK");

  let afkActive = false;

  pi.registerCommand("afk", {
    description: "Enter AFK autonomous mode",
    handler: (_args, ctx) => {
      if (afkActive) {
        ctx.ui.notify("[afk] already active — use /back to return", "info");
        return;
      }
      afkActive = true;
      ctx.ui.setStatus("afk", "AFK 🔴");
      ctx.ui.notify("[afk] engaged", "info");
      pi.sendUserMessage(AFK_PROMPT, { deliverAs: "followUp" });
    },
  });

  pi.registerCommand("back", {
    description: "Return from AFK",
    handler: (_args, ctx) => {
      if (!afkActive) {
        ctx.ui.notify("[afk] not active", "info");
        return;
      }
      afkActive = false;
      ctx.ui.setStatus("afk", "");
      ctx.ui.notify("[afk] disengaged — welcome back", "info");
      pi.sendUserMessage(BACK_PROMPT, { deliverAs: "followUp" });
    },
  });
}
```

`deliverAs: "followUp"` fires the message immediately when the agent is idle (which it is when you type a slash command). The handler returns, the command pipeline closes, and the injected message kicks off the next agent turn.

`ctx.ui.setStatus(key, text)` writes to the OMP status bar. The key is stable — passing an empty string clears it. This is the same mechanism `agent-slowmo` uses to show the per-tool countdown.

---

## The AFK prompt

The injected prompt is the content that was previously in the markdown command file, verbatim:

```
The user is AFK. Switch to autonomous mode with these constraints:

AUTONOMOUS MODE RULES:
- Blast radius: LOW only — edits, reads, local commits are fine.
  No git push, no external API calls that create/delete resources.
- Work from the current todo list — pick the next actionable P1/P2
  item that doesn't require user input.
- If stuck: write a note to ~/tmp/afk-notes.md and stop.
  Do NOT loop trying variants.
- When done: write a summary to ~/tmp/afk-done.md.

BEFORE STARTING:
1. Read ~/.omp/agent/daily-goal.md
2. Read ~/Notes/todo.md — pick the highest-priority actionable item
3. State: "AFK mode: working on [X]. Will stop if [condition]."

Proceed now.
```

The `/back` message is minimal: `User is back. Exit AFK autonomous mode. Stop at the next clean checkpoint. Wait for my next instruction.`

---

## Slash commands

| Command | Effect |
|---|---|
| `/afk` | Engage autonomous mode, inject AFK prompt, show status |
| `/back` | Disengage, notify agent, clear status |

State is in-memory per session. Restarting OMP resets it (which is correct — you wouldn't want a stale AFK flag persisting across sessions).

---

## What didn't work

**The markdown command.** A `.claude/commands/afk.md` file can inject the prompt, but it has no handler — just static text expansion. No guard against double-invocation, no status bar, no `/back` counterpart. It works for one-shot use but not as a toggle.

**State in a file.** Writing a flag to `~/.omp/agent/afk-state` on each toggle would let the extension restore AFK state on session resume. In practice this creates more problems than it solves: if OMP crashes mid-AFK, the stale flag re-engages on restart with no context. In-memory state that starts clean every session is the right default for something this transient.

---

## When to use it

- You need the agent to continue unattended while you're in a meeting, on a call, or away from the machine
- The task is well-defined and queued in `todo.md`
- Blast radius is known and low (no pushes, no external writes, no irreversible ops)

Flip it off with `/back` the moment you return — don't let the agent keep running while you're reading its output.

---

## Source

- [agent-afk-omp](https://github.com/ankitg12/agent-afk-omp) — the extension (`agent-afk.ts`, ~50 lines of TypeScript)
- [Slow-motion mode]({% post_url 2026-05-11-slowmo-for-ai-coding-agents %}) — the `ctx.ui.setStatus` pattern comes from here
- [Personal Agentic Process]({% post_url 2026-05-06-personal-agentic-process %}) — the broader discipline this fits into
