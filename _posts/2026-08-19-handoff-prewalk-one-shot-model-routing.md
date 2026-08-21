---
layout: post
title: "Give a Handoff One Planning Turn, Then Switch Back"
date: 2026-08-19
categories: ai agents productivity
series: "AI coding agent productivity"
---

A coding-agent handoff often needs the strongest reasoning at the beginning, not for the rest of the session.

The child session has to reconstruct intent, inspect inherited state, and decide where to cut into the code. Once it makes the first successful mutation, much of that ambiguity is gone. Keeping the planning model active after that boundary adds latency and cost to routine implementation work.

I built an OMP extension that treats a handoff as a one-shot model-routing lifecycle:

```text
/handoff
    │
    ▼
switch child to \@slow
    │
    ├── read/search/bash ── stay on \@slow
    ├── failed edit/write ─ stay on \@slow
    │
    ▼
first successful edit or write
    │
    ▼
switch to \@smol and disarm
```

The result is one strong planning turn followed by a fast implementation model. Neither concrete model is hard-coded.

## What I borrowed—and changed

The underlying idea comes from Can Bölük's [Prewalk](https://stencil.so/blog/prewalk): let a frontier model explore, establish a trajectory, and land the first edit; then let a cheaper model continue from evidence rather than from a read-only plan. His SWE-Bench Pro results show why this is more interesting than a conventional planner/executor split: the first edit is proof that the approach has touched the code.

This extension adapts that idea specifically to OMP handoffs. It resolves role aliases instead of fixed models, arms only on `session_switch(reason: "handoff")`, exposes the pending transition in the status line, and records the lifecycle in JSONL.

There is also an important difference. The original Prewalk couples the first edit with a concrete TODO checklist that keeps the executor on trajectory. This narrower extension currently uses the first successful `edit` or `write` as its only boundary. That is a deliberate simplification, not an equivalent benchmark claim. If live traces show shallow or premature transfers, requiring an active checklist is the next guard to add.

---


## Why handoffs need their own lifecycle

A normal new session and a handoff do not start from the same state.

| Session type | Initial problem |
|---|---|
| New session | Discover the task and relevant repository context |
| Handoff | Reconstruct another session's intent, progress, constraints, and stopping point |

The handoff child inherits a narrative, but inheritance is not understanding. Its first useful action is usually synthesis: determine what is already proven, what remains open, and which next mutation is safe.

That is where a planning model earns its keep. After the first successful mutation, the session has crossed from reconstruction into execution.

The boundary should therefore be an observed event, not a timer or token estimate.

---

## Resolve roles, not model IDs

The extension uses settings-backed model roles:

```ts
const PLANNER_ROLE = "@slow";
const TARGET_ROLE = "@smol";

const planner = ctx.models.resolve(PLANNER_ROLE);
const target = ctx.models.resolve(TARGET_ROLE);
```

Configuration owns the concrete mapping. The extension owns only the lifecycle.

This separation has three useful properties:

1. Providers can change without editing extension code.
2. Project or global settings remain the source of truth.
3. The same policy works across machines and accounts.

Hard-coding model IDs would turn a reusable extension into a personal configuration file.

---

## Arm only after the planner switch succeeds

A handoff first clears stale state, resolves both roles, and switches to the planner. The one-shot is armed only when that switch reports success.

```ts
pi.on("session_switch", async (event, ctx) => {
  disarm();
  ctx.ui.setStatus(STATUS_KEY, undefined);

  if (event.reason !== "handoff") return;

  const roles = resolveRoles(ctx);
  if (!roles) return;

  const myGeneration = generation;
  const switched = await pi.setModel(roles.planner);
  if (generation !== myGeneration || !switched) return;

  armed = { target: roles.target, generation: myGeneration };
  ctx.ui.setStatus(STATUS_KEY, `Pre-walk → ${TARGET_ROLE}`);
});
```

The status line describes the pending transition. `Pre-walk → \@smol` means the planner is active and the extension is waiting for the execution boundary. The entry disappears after the target switch succeeds.

The generation counter matters because model switching is asynchronous. A result from an old session switch must not arm a newer session accidentally.

---

## Define the execution boundary narrowly

Not every tool call means planning is finished.

| Tool result | Consume the one-shot? |
|---|---:|
| Read, search, or shell inspection | No |
| Failed `edit` | No |
| Failed `write` | No |
| Successful `edit` | Yes |
| Successful `write` | Yes |

The demotion handler checks all three conditions:

```ts
const DEMOTION_TOOLS: Record<string, true> = {
  edit: true,
  write: true,
};

pi.on("tool_execution_end", async (event, ctx) => {
  const pending = armed;
  if (
    !pending ||
    pending.generation !== generation ||
    event.isError ||
    !DEMOTION_TOOLS[event.toolName]
  ) return;

  armed = undefined;
  const switched = await pi.setModel(pending.target);

  if (!switched) {
    armed = pending;
    return;
  }

  ctx.ui.setStatus(STATUS_KEY, undefined);
});
```

Claiming the one-shot before awaiting `setModel` prevents parallel tool completions from switching twice. If the switch fails, restoring `armed` makes the next successful mutation a retry rather than silently abandoning the policy.

---

## Durable logs make the lifecycle observable

A model indicator shows the current state. It does not explain how the session got there.

The extension appends JSONL records to `~/.omp/agent/handoff-prewalk.log`:

```json
{"event":"loaded","plannerRole":"@slow","targetRole":"@smol"}
{"event":"session_switch","reason":"handoff","generation":1}
{"event":"planner_switch_started","model":"provider/planner-model"}
{"event":"planner_switch_finished","switched":true}
{"event":"armed","target":"provider/fast-model"}
{"event":"demotion_triggered","toolName":"edit"}
{"event":"demotion_finished","switched":true,"target":"provider/fast-model"}
```

That sequence distinguishes failures which otherwise look identical in the UI:

- the extension did not load;
- one role did not resolve;
- both roles resolved to the same model;
- the planner switch failed;
- the extension armed but no successful mutation occurred;
- the target switch failed and remains eligible for retry.

Notifications are feedback. Logs are evidence.

---

## Test the state machine, then prove it live

Unit tests cover the local invariants:

- handoff switches to the configured planner;
- the first successful `edit` or `write` switches to the target;
- read-only and failed tools do not consume the one-shot;
- a failed target switch retries later;
- unrelated session switches disarm the state;
- the status appears while armed and clears afterward.

But extensions load at process startup, so passing tests are not the final proof. A live check requires a full OMP reload followed by a real `/handoff`.

The acceptance sequence is:

1. Confirm a new `loaded` record names `\@slow` and `\@smol`.
2. Run `/handoff`.
3. Confirm the planner model and `planner_switch_finished` record.
4. Confirm `Pre-walk → \@smol` in the status line.
5. Perform one successful `edit` or `write`.
6. Confirm the target model and `demotion_finished` record.
7. Confirm the Pre-walk status is gone.

A passing test proves the state machine. A live handoff proves the extension is loaded, connected to real events, and controlling the actual session model.

---

## The reusable pattern

This extension is a small example of event-driven model routing.

Instead of assigning one model to an entire session, assign models to phases with observable boundaries:

| Boundary | Routing policy |
|---|---|
| Handoff opened | Use a planner |
| First successful mutation | Return to a fast model |
| Repeated tool failure | Escalate temporarily |
| Review requested | Switch to a reviewer |
| Review completed | Return to implementation |

The hard part is not calling `setModel`. The hard part is choosing a transition that is narrow, observable, retryable, and easy to diagnose.

For handoffs, the first successful mutation is a useful boundary: it marks the point where inherited context became concrete action.

## Source

- [Prewalk: “You only need the frontier model for one single edit”](https://stencil.so/blog/prewalk)
- [Oh My Pi](https://github.com/can1357/oh-my-pi)
