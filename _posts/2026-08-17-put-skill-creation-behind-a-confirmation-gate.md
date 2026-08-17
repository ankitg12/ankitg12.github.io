---
layout: post
title: "Put Agent Skill Creation Behind a Confirmation Gate"
date: 2026-08-17
categories: ai agents productivity
series: "AI coding agent productivity"
---

An agent that turns every useful observation into a reusable skill eventually makes every future session pay for yesterday's enthusiasm.

In one agent setup, the always-loaded catalog had grown to 103 skill descriptions. Most were individually reasonable. Together they occupied 18,715 prompt characters before the agent had read the task.

This is an admission-control problem, not a summarization problem.

## Notes and skills have different jobs

A durable observation should survive, but it does not automatically deserve executable packaging.

| Capture as | Use for | Example |
|---|---|---|
| Note | Fact, preference, correction, decision, or project convention | "This API reports UTC timestamps." |
| Skill | Repeatable multi-step procedure with a recognizable future trigger | "When a release fails signature verification, run these checks in order." |

A note is cheap to store and retrieve. A skill adds a name, trigger description, procedure, maintenance surface, and often an entry to an always-loaded catalog. Treating the two as interchangeable produces skill sprawl.

A practical admission test is:

> Can I name a plausible future request that should trigger this exact multi-step procedure?

If not, save a note.

## Intercept the write, not the model's intention

Prompt guidance helps, but it is advisory. A stronger boundary sits immediately before the skill-creation tool executes.

```ts
pi.on("tool_call", async (event, context) => {
  if (event.toolName !== "manage_skill") return;

  const input = event.input ?? {};
  if (input.action !== "create") return;

  if (!context.hasUI) {
    return {
      block: true,
      reason: "Skill creation requires explicit approval.",
    };
  }

  const approved = await context.ui.confirm(
    "Create managed skill?",
    [
      `Create '${input.name}'?`,
      "Prefer a note for facts, conventions, and corrections.",
      "Use a skill only for a repeatable multi-step procedure",
      "with a plausible future trigger.",
    ].join("\n"),
  );

  if (!approved) {
    return {
      block: true,
      reason: "Save this as a note instead.",
    };
  }
});
```

The boundary is deliberately narrow:

- creation requires confirmation;
- updates and deletion continue unchanged;
- unrelated tools continue unchanged;
- non-interactive sessions fail closed.

This is human-in-the-loop approval at the capability boundary. The agent can still propose a reusable procedure, but it cannot silently increase the permanent prompt surface.

## The first design was safer-looking and worse

The initial implementation used two steps:

1. Block the first creation attempt and return the note-versus-skill test.
2. Ask the agent to repeat the identical call, then show a user confirmation dialog.

The retry carried a fingerprint of the proposed payload so approval could not be reused for modified content. That was defensible, but it solved a problem the UI confirmation already solved.

| Design | User actions | Agent behavior | State required |
|---|---:|---|---|
| Block, retry, confirm | 1 confirmation after an extra agent turn | Must reproduce the call exactly | Pending fingerprint and retry state |
| Confirm immediately | 1 confirmation | No retry protocol | None |

The two-step design added ceremony without adding a meaningful safety boundary. The hook already runs before execution and already has the exact payload. Asking on the first call is both simpler and clearer.

The revised flow is:

```text
agent proposes create
        |
        v
show note-vs-skill test and ask user
        |
   +----+----+
   |         |
approve    decline
   |         |
create    block; suggest note
```

This pivot removed the pending token, payload fingerprint, concurrency guard, and retry instructions. Fewer states also meant fewer failure modes.

## Global is a placement decision

Passing the admission test does not mean a skill belongs in the global catalog.

| Scope | Put the skill here when |
|---|---|
| Global | Unrelated agents can recognize the trigger and execute the procedure |
| Agent-specific | The procedure depends on one agent's role, tools, or operating contract |
| Project-specific | The procedure depends on one repository, testbed, or project vocabulary |

A specialized procedure can be an excellent skill and still be a poor global skill. Moving it to the agent or project that owns the trigger preserves the automation without charging every unrelated session for its description.

This gives skill admission two gates:

1. **Form:** is this a repeatable procedure rather than a note?
2. **Scope:** which sessions can plausibly trigger it?

## Use content-to-description ratio as a smell

Another useful signal is how much procedure exists beyond the catalog description:

```text
content ratio = skill body characters / description characters
```

If the body barely expands the description, packaging it as a skill buys little. In an inventory of 88 managed skills, only one fell below `3x` and two fell below `5x`; the smallest was also independently classified as a fact better kept in a tooling note.

The ratio is a review trigger, not an automatic deletion rule. A short safety-critical sequence may still deserve a skill, while a long file can be mostly history or prose. Use it to find candidates, then apply the form and scope tests.

## Test the policy at the boundary

The focused test matrix is small:

| Case | Expected result |
|---|---|
| First create, approved | Confirmation appears; call proceeds |
| First create, denied | Confirmation appears; call is blocked |
| Create without UI | Call is blocked without prompting |
| Update or delete | Call passes through |
| Any other tool | Call passes through |

A live probe matters in addition to unit tests. Invoke creation with a disposable name, decline it, and verify the authoritative skill directory remains absent. Unit tests prove handler logic; the probe proves registration and runtime wiring.

## Reduce existing catalog cost separately

Admission control prevents new growth. It does not curate what is already there.

Measure the catalog first, then keep only skills relevant to the agent's standing role. In this case, filtering unrelated skill families changed the prompt as follows:

| Metric | Before | After | Change |
|---|---:|---:|---:|
| Loaded skills | 103 | 45 | -58 |
| Skill catalog characters | 18,715 | 8,438 | -54.9% |
| Whole prompt characters | 54,623 | 44,346 | -18.8% |

The instruction files were byte-for-byte unchanged. That matters: the reduction came from removing irrelevant capability advertisements, not weakening project rules.

The durable pattern is therefore two-part:

1. Curate the existing always-loaded catalog by role.
2. Put explicit admission control in front of every new reusable skill.

Notes preserve learning. Skills package behavior. Keeping that distinction sharp makes both easier to trust.

## Source

- [OpenAI Agents SDK: human-in-the-loop](https://openai.github.io/openai-agents-python/human_in_the_loop/)
- [LangChain: human-in-the-loop middleware](https://reference.langchain.com/javascript/langchain/index/humanInTheLoopMiddleware)
- [Microsoft Agent Framework: tool approval](https://learn.microsoft.com/en-us/agent-framework/agents/tools/tool-approval)
