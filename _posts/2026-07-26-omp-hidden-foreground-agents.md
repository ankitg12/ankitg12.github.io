---
layout: post
title: "OMP's hidden foreground agents: /tan, Agent Hub, and the ownership trap"
date: 2026-07-26
categories: ai agents productivity
series: "AI coding agent productivity"
---

I recently asked what seemed like a missing-feature question: *OMP can launch tangential work with `/tan`, but can I bring that agent into the foreground, talk to it directly, and still let the main agent coordinate with it?*

The answer was already installed. It was hidden across two features whose names suggest different workflows:

```text
/tan Investigate why this search deduplicator missed a repeat
Alt+A
select Tan-* → Enter
```

`/tan` creates the agent. **Agent Hub** makes its live transcript interactive. The combination is effectively a foreground tangent—but understanding it requires separating three ideas that terminal-agent interfaces often blur:

- **Execution:** which agent is actually running?
- **View:** which transcript is currently on screen?
- **Ownership:** which agent is responsible for changing the artifact?

Background is an execution state. Foreground is UI focus. Ownership is a workflow decision. Confusing the third with either of the first two is where things get interesting.

## The feature arrived in two pieces

OMP added `/tan` in [v15.10.1](https://github.com/can1357/oh-my-pi/releases/tag/v15.10.1) on June 7, 2026. The release note accurately calls it a way to fork the current conversation into a background agent while the main session stays active.

Three days later, [v15.11.0](https://github.com/can1357/oh-my-pi/releases/tag/v15.11.0) added Agent Hub:

- `Alt+A`, `Ctrl+S`, or double-tap left arrow on an empty editor opens it;
- the table shows registered agents, status, activity, and unread messages;
- `Enter` opens a full-screen transcript;
- input can steer a running agent;
- persistent `task` agents can remain idle, park to disk, and later revive.

The public homepage advertises first-class subagents, and the release notes describe both components, but the operational sentence is easy to miss:

> Start with `/tan`; foreground and steer the running tangent through Agent Hub.

There is no separate `/foreground-agent` command because foreground is a view over an existing agent, not another kind of agent.

## What `/tan` actually inherits

A tangent is not a fresh blank session. The implementation first flushes the parent session, then forks its session history. It creates a new agent session with the parent's:

- conversation history up to the fork point;
- system prompt;
- model and thinking level;
- active tools and settings;
- working directory and parent artifact directory;
- shared MCP proxy tools;
- provider prompt-cache lineage.

OMP then makes two deliberate changes.

First, it clears the copied todo list. Without that, reminders inherited from Main would pull the tangent back toward the parent's task.

Second, it injects a developer message establishing a context switch: the previous conversation belongs to the parent; the tangent must focus exclusively on the newly supplied work. OMP reinjects that boundary after compaction so a summary cannot blur the parent task back into the tangent's mission.

So `/tan` has an unusually useful context contract:

```text
shared past + separate present + explicitly narrowed mission
```

After the fork, the histories diverge. New Main messages do not magically appear in the tangent, and tangent messages do not become part of Main's transcript. Results and explicit Hub communication cross that boundary.

## What “foreground” means

Opening Agent Hub does not suspend Main, migrate the tangent, or merge histories. It changes which transcript the TUI presents and where your next input goes.

```text
                    shared registry / messaging
                  ┌────────────────────────────┐
                  │                            │
            ┌─────▼─────┐                ┌─────▼─────┐
            │   Main    │                │  Tan-*    │
            │ transcript│                │ transcript│
            └───────────┘                └───────────┘
                  ▲                            ▲
                  └────── Agent Hub view ─────┘
```

While the tangent is running, opening its transcript and submitting text steers its active turn. Main can independently communicate through OMP's shared Hub/messaging broker. Pressing `Esc` returns to the hub or Main view; it does not terminate the tangent.

A separately launched `omp` process in another terminal is different. It is a new top-level session with a separate in-process agent registry and broker. It gives visual concurrency, but not the same native Main↔tangent coordination.

## One lifecycle subtlety: `/tan` is not `task`

Agent Hub displays both, but their completion semantics differ:

| Agent kind | While running | After successful completion |
|---|---|---|
| `/tan` fork | Live transcript; steer through Agent Hub | Deliberately detached as a transcript-only parked agent; not revivable |
| Persistent `task` agent | Live transcript; Hub/agent messaging | Stays idle, later parks with a reviver, and can be awakened by another message |

This distinction matters. The generic Agent Hub UI advertises `r:revive`, but a completed `/tan` has no live session or reviver attached. Its transcript remains inspectable; continuing substantial work requires a new tangent or a persistent task agent.

Use `/tan` for one bounded side investigation that should inherit the conversation. Use `task` when you expect a longer-lived collaborator with follow-up turns.

## The ownership trap we hit immediately

We used `/tan` to diagnose why a knowledge-gem search had repeated the same articles. The tangent found the real defect, updated the tool, added regression tests, cleaned duplicate rows, and returned a verified result.

Then Main reviewed the result—and continued editing the related blog post. Nothing in OMP had silently switched agents. The human had switched UI focus, the tangent had finished, and Main simply continued the thread. But the experience *felt* like the work had leaked back into Main because we had never declared ownership explicitly.

This is a coordination problem, not a process problem.

A reliable delegation sentence is:

```text
Tangent owns implementation. Main may monitor, review, and report,
but must not edit unless ownership is explicitly handed back.
```

The inverse is also useful:

```text
Tangent investigates and recommends only. Main owns every edit.
```

Without that contract, both agents can act reasonably and still duplicate or interleave work. Agent Hub solves observability and communication; it cannot infer project ownership from whichever transcript happens to be visible.

## A practical operating protocol

### 1. Declare the boundary in the `/tan` request

Bad:

```text
/tan Look into the dedupe problem
```

Better:

```text
/tan Own the dedupe fix end to end: diagnose, edit, and run focused
verification. Main will only review and report; message Main if blocked.
```

### 2. Open Agent Hub while the tangent is running

Press `Alt+A`, select the `Tan-*` row, and press `Enter`. The status and transcript—not your memory of the last screen—tell you which agent is active.

### 3. Steer outcomes, not keystrokes

Send corrections such as:

```text
The real contract is repeated knowledge, not repeated query wording.
Check whether result URLs overlap before changing the fuzzy threshold.
```

This preserves the tangent's agency while correcting the problem definition.

### 4. Treat completion as a handoff point

When the tangent yields, Main should do one of three things explicitly:

- **accept and report**;
- **reject and launch another bounded tangent**;
- **take ownership back**, stating that Main will now edit.

“Continue naturally” is precisely the ambiguous fourth state to avoid.

## Why this deserves documentation

The underlying implementation is thoughtful: session cloning preserves useful context, the todo reset prevents task contamination, the context-switch message survives compaction, and Agent Hub exposes live supervision without collapsing independent histories.

The discovery path is weaker. `/tan` describes background execution, while Agent Hub is exposed through a keyboard shortcut. Unless a user reads two release notes or happens upon the welcome-screen tip, there is little reason to infer that the two compose into an interactive foreground workflow.

A small upstream affordance would close most of the gap:

```text
Dispatched Tan-153e… — press Alt+A to observe or steer while it runs
```

That message teaches the mental model at the moment it becomes useful.

## Takeaways

- `/tan` is a context-rich fork, not a fresh agent.
- Agent Hub foregrounds the transcript; it does not merge or migrate sessions.
- Main and subagents can coordinate through the shared registry and messaging broker.
- A completed `/tan` is transcript-only; persistent follow-up belongs to a `task` agent.
- UI focus is not task ownership. Declare ownership before dispatch and at handoff.
- Good agent orchestration requires both mechanisms and protocols: the harness provides communication, while the team provides accountability.

The deeper lesson is broader than OMP: **multi-agent systems fail less often from a lack of intelligence than from ambiguous authority.** The moment several capable actors can see and modify the same world, ownership becomes part of the technical architecture.

## Source

- [OMP v15.10.1 release notes — `/tan`](https://github.com/can1357/oh-my-pi/releases/tag/v15.10.1)
- [OMP v15.11.0 release notes — Agent Hub and persistent agents](https://github.com/can1357/oh-my-pi/releases/tag/v15.11.0)
- [`tan-command-controller.ts`](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/modes/controllers/tan-command-controller.ts)
- [`agent-hub.ts`](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/modes/components/agent-hub.ts)
- [OMP `task` internals and lifecycle](https://github.com/can1357/oh-my-pi/blob/main/docs/tools/task.md)
