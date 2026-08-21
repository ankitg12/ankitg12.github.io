---
layout: post
title: "Put Subagent Spawns Behind an Approval Gate"
date: 2026-08-19
categories: ai agents productivity
series: "AI coding agent productivity"
---

A subagent can be useful and still be the wrong call.

Delegation adds latency, cost, and another context boundary. A coding agent may spawn a reviewer or scout because parallelism is available, not because it materially improves the task. By the time the user sees the extra work, the process is already running.

I wanted a narrower policy: keep subagents available, but require explicit approval immediately before OMP's canonical `task` tool spawns them.

```text
agent proposes task call
        │
        ▼
subagent-admission summarizes the spawn
        │
        ▼
user approves? ── no ── block tool call
        │
yes     ▼
        run task normally
```

This follows the same capability-boundary pattern as {% post_url 2026-08-17-put-skill-creation-behind-a-confirmation-gate %}: do not remove a useful capability; put the consequential transition behind consent.

---

## Why prompting before the spawn matters

An after-the-fact notification is not approval.

| Control point | What the user can still decide |
|---|---|
| Before tool execution | Whether the subagent should exist at all |
| After process start | Whether to interrupt work already in progress |
| After completion | Nothing; cost and latency are already spent |

The correct interception point is OMP's `tool_call` event. It runs after the model has proposed a tool invocation but before the tool executes.

The extension ignores every tool except `task`:

```ts
pi.on("tool_call", async (event, context) => {
  if (event.toolName !== "task") return undefined;

  // Admission policy follows here.
});
```

That makes the policy precise. Reads, edits, shell commands, and other tools continue unchanged.

---

## Show what is about to spawn

Approval without enough context becomes a reflexive Yes button.

The dialog includes three fields for every proposed subagent:

- stable name;
- agent type;
- compact task summary.

A batch receives one consolidated prompt rather than one modal per child:

```text
Spawn 2 subagents?

- ScoutCode [scout]: Inspect the relevant code paths.

- ReviewRisk [reviewer]: Review the proposed change for safety risks.

Subagents add latency and cost. Approve only when delegation materially helps.
```

The summary normalizes whitespace and caps task previews at 240 characters. That is enough to identify the work without turning the confirmation dialog into a transcript viewer.

```ts
function shortTask(value: unknown): string {
  if (typeof value !== "string") return "(task not provided)";

  const compact = value.replace(/\s+/g, " ").trim();
  if (!compact) return "(task not provided)";

  return compact.length > 240
    ? `${compact.slice(0, 237)}...`
    : compact;
}
```

Defaults also matter. If a caller omits a name or agent type, the dialog displays `spawn 1` and `default` rather than hiding the missing information.

---

## Approval returns no override; denial blocks

OMP's pre-tool hook uses a small decision object. Returning `undefined` lets the original tool call continue. Returning `{ block: true, reason }` prevents execution and gives the agent a reason it can act on.

```ts
const approved = await context.ui.confirm(
  "Approve subagent spawn?",
  approvalMessage(spawns),
);

if (!approved) {
  return {
    block: true,
    reason:
      "User did not approve the subagent spawn. " +
      "Continue in the current agent or ask again only if delegation is necessary.",
  };
}

return undefined;
```

The denial reason is part of the policy. It tells the current agent to continue locally instead of immediately presenting the same dialog again.

This is admission control, not a warning banner.

---

## Non-interactive execution must fail closed

A confirmation policy cannot quietly become permissive when no UI exists.

```ts
if (!context.hasUI) {
  return {
    block: true,
    reason:
      "Spawning subagents requires explicit user approval " +
      "in an interactive OMP session.",
  };
}
```

Failing closed has an operational cost: unattended workflows using `task` cannot spawn children while this extension is active. That is intentional. A future allowlist could grant specific automation paths, but absence of a UI is not consent.

---

## Be explicit about the coverage boundary

The extension guards OMP's canonical `task` tool, including single and batch task invocations. That is the normal model-facing route for spawning subagents.

It does not scan arbitrary JavaScript passed to an evaluation tool looking for `agent(...)`. Source-code scanning would be brittle: aliases, generated calls, wrappers, and innocent strings all make text matching unreliable.

| Spawn surface | Covered? | Reason |
|---|---:|---|
| `task` tool, single child | Yes | Canonical pre-tool event |
| `task` tool, batch children | Yes | One prompt lists the full batch |
| Non-interactive `task` call | Blocked | No UI means no approval |
| Nested `agent()` inside arbitrary evaluation code | No | Separate bridge; text scanning is not a reliable security boundary |

A complete platform-wide gate would require a shared pre-spawn hook below both execution paths. Until that hook exists, precise coverage is better than pretending a regex is enforcement.

---

## Test both the policy and the live dialog

The focused tests cover six cases:

1. unrelated tools pass without prompting;
2. a single spawn shows its name, type, and task;
3. denial blocks the tool call;
4. a batch gets one prompt listing every child;
5. non-interactive calls fail closed;
6. missing fields and long summaries remain readable.

The live test is even simpler:

1. Reload OMP, because extensions are not hot-reloaded.
2. Ask the agent to create one harmless task.
3. Confirm the approval dialog appears before any child starts.
4. Deny it.
5. Verify the tool returns the policy reason and no subagent runs.

That denial path is the most important proof. A confirmation dialog that appears after the child starts is decoration, not control.

---

## The broader rule

Agent systems accumulate capabilities faster than they accumulate judgment about when to use them.

The answer is not to ask for confirmation before every tool. That produces habituation and makes the interface worse. Approval gates belong at narrow, consequential boundaries:

- create a durable skill;
- spawn another agent;
- publish or push externally;
- delete data;
- cross an authentication or billing boundary.

The implementation can be small because the policy is specific. Intercept one canonical event, present enough context for a real decision, fail closed when consent is impossible, and return a reason the agent can follow.

## Source

- [subagent-admission source](https://github.com/ankitg12/omp-extensions/tree/master/packages/subagent-admission)
- [Oh My Pi](https://github.com/can1357/oh-my-pi)
