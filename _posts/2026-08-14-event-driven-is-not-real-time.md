---
layout: post
title: "Event-Driven Is Not Real-Time"
date: 2026-08-14
categories: ai agents productivity terminal
series: "AI coding agent productivity"
---

A focus-change hook that pauses the agent I just left sounds deterministic until the hook takes six seconds to decide what “focused” means.

I wanted a simple safety invariant for several terminal coding agents:

> Only the agent in the currently focused pane may run. Every other agent remains paused until I resume it manually.

The first implementation was event-driven. A terminal multiplexer emitted focus events, a plugin listed the agents, found the background panes, inspected their screens, and submitted the coding agent's native `/pause` command.

Every component worked. The system did not.

## The first false proof

Unit tests covered the obvious policy:

| Pane state | Focused? | Action |
|---|---:|---|
| working | yes | skip |
| working | no | pause |
| idle | no | pause |
| already paused | no | skip |
| not a coding agent | no | skip |

A scripted focus command also produced the expected plugin log:

```text
pause-sent pane=p1
```

Neither result proved the human interaction. During real UI switching, both agents sometimes continued running. On other attempts, an agent paused tens of seconds later.

The important evidence was not whether `/pause` eventually appeared. It was the timeline:

```text
focus event emitted
    ↓
plugin process starts
    ↓
list agents
    ↓
read candidate screen
    ↓
recheck focus
    ↓
submit /pause
    ↓
agent reaches a safe boundary
    ↓
pause overlay appears
```

“Event-driven” describes how work starts. It says nothing about how long the path takes.

## Three latencies, not one

The visible delay contained three different mechanisms.

### 1. Event-handler latency

Each focus event launched a plugin subprocess. The subprocess then made several command-line round trips back to the terminal multiplexer.

A single UI switch emitted workspace, tab, and pane focus events. Subscribing to all three created three concurrent handlers for one human action.

### 2. Reconciliation latency

The handler used a snapshot:

```python
agents = list_agents()
for agent in agents:
    if not agent.focused:
        pause(agent)
```

Focus could change after `list_agents()` but before `pause(agent)`. A just-before-submit recheck prevented pausing the newly focused pane, but it exposed the opposite failure: the pane just left might never be paused.

This is a time-of-check to time-of-use race. The policy is about *current* focus, while the implementation acts on an aging observation.

### 3. Agent safe-boundary latency

The native `/pause` command does not terminate an in-flight tool call. The current call may finish, after which the agent enters its pause overlay before starting new work.

That delay is intentional. It is different from a plugin that has not submitted `/pause` yet.

The logs therefore need to distinguish:

```text
handler started
focus state read
pause submitted
pause acknowledged
pause visible
```

A total handler duration cannot identify which boundary caused the delay.

## Why a lock made it worse

The obvious response to duplicate handlers was a file lock. Only one handler would reconcile; the others would coalesce.

The first version dropped overlapping events. If focus changed while one reconciliation was active, the latest state might never receive a final pass.

The second version made every waiter acquire the lock eventually. That preserved events but converted a burst into a queue:

```text
event A ── reconcile old state
          event B ── wait ── reconcile older state
                    event C ── wait ── reconcile stale state
```

Correct serialization is not the same as current-state correctness. A safety action performed reliably on stale state is still wrong.

A better coalescing primitive needs both properties:

1. **single flight** — at most one expensive reconciliation runs at a time;
2. **trailing edge** — if an event arrives during that run, exactly one final reconciliation uses the newest state.

Even that does not solve a six-second control path. Debouncing duplicate work cannot turn slow observation into real-time enforcement.

## One switch should produce one hook

The first useful simplification was to remove redundant focus subscriptions. Real logs showed that a workspace switch also emitted `pane.focused`, so the plugin retained only the narrow event needed by the policy.

This reduced amplification:

```text
before: workspace.focused + tab.focused + pane.focused
 after: pane.focused
```

It did not make the remaining command-line round trips instantaneous. Reducing the number of slow paths is not the same as building a fast path.

## The design rule

A real-time safety invariant cannot depend on a slow asynchronous observer.

If pausing must follow focus immediately, the control should live as close as possible to the focus transition:

```text
focus owner
    ├── changes focused pane
    └── applies pause policy from the same authoritative state
```

If the plugin contract cannot support that atomicity, the honest options are:

| Mode | Claim |
|---|---|
| Integrated control path | Deterministic focus safety |
| Asynchronous best effort | Background agents will usually pause later |
| Disabled | No automated protection |

The interface must name its guarantee correctly. A delayed best-effort mechanism may still be useful, but it must not be presented as a lock.

## Verification gates

A useful test plan must cross the process and UI boundaries:

1. Start with two unpaused agents.
2. Switch through the real UI, not a surrogate focus command.
3. Record the focus-event timestamp.
4. Record each state-read and command-submission timestamp.
5. Confirm the pane just left receives `/pause`.
6. Return to that pane and confirm the native pause overlay is still present.
7. Resume it manually.
8. Repeat rapidly enough to overlap handlers.
9. Test while one agent has an in-flight tool call.
10. Fail the test if either agent starts new work after its safe boundary.

Unit tests remain valuable. They prove selection rules and race guards. They do not prove that event dispatch, subprocess startup, command transport, terminal state, and agent control compose within the required time.

The general lesson extends beyond coding agents: **an event is notification that reality changed, not a transaction that changed reality safely.**

## Source

- [Herdr — runtime for coding agents](https://github.com/herdrdev/herdr)
- [Oh My Pi — terminal coding agent](https://github.com/can1357/oh-my-pi)
- [Ambient Attention Without Interruption]({% post_url 2026-08-11-ambient-attention-without-interruption %})
