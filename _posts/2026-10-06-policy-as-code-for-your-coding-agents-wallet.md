---
layout: post
title: "Policy-as-Code for Your Coding Agent's Wallet"
date: 2026-10-06 10:45:00 +0530
categories: ai agents productivity
series: "AI coding agent productivity"
---

The most expensive model is the right choice for the first ten minutes of a session, and the wrong choice for the next two hours.

Yesterday's post, [Decaying Effort]({% post_url 2026-10-05-decaying-effort-transmission-gearbox-ai-agents %}), changed *how hard* one model thinks as a session ages. That leaves the bigger lever untouched: *which* model is thinking. Once the design is settled and the agent is writing tests and renaming fields, a frontier model is a waste of money. But "switch models after a while" is not a policy. Different people want different triggers:

- "After $1 of spend, drop from the big model to the mid model."
- "Only do that if I am actually on the big model."
- "Never do it in subagents."
- "After 40 turns, go to the cheap model, but at medium effort."

Hard-coding any one of these gives you a tool that fits one person. What you need is a small rule language.

## Why CEL

[CEL (Common Expression Language)](https://cel.dev) is Google's expression language for exactly this job: evaluating a policy against a set of facts. Kubernetes uses it in `ValidatingAdmissionPolicy`, and Envoy and Google Cloud IAM use it for conditions. Three properties make it a good fit here:

| Property | Why it matters for an agent policy |
|---|---|
| Not Turing-complete, no side effects | A rule cannot loop, hang the session, or touch the file system |
| Type-checked before evaluation | `costt > 1` (a typo) fails when the config loads, not silently at turn 40 |
| Familiar syntax | `cost > 1.0 && model.startsWith("big/")` reads like the intent |

The alternatives were worse. A home-made "mini grammar" of `>` and `&&` grows, one feature at a time, into a worse CEL. Rego (OPA) is powerful, but a whole policy engine to compare two numbers is overkill.

## The extension

`model-shift-omp` is an extension for the [oh-my-pi](https://github.com/can1357/oh-my-pi) coding agent. Rules go in `~/.omp/model-shift.yml`:

```yaml
rules:                    # ordered: first match wins
  - name: budget
    when: 'cost > 1.0 && model.startsWith("anthropic/claude-opus")'
    use: '@smol'          # a provider/id, or a role alias
  - name: long-session
    when: 'turns >= 40'
    use: anthropic/claude-sonnet-4-5
    effort: medium
```

These are the facts a rule can use:

| Variable | CEL type | Meaning |
|---|---|---|
| `cost` | double | Session spend in USD, including subagents |
| `tokens` | int | Current context tokens |
| `context_pct` | double | Percentage of the context window used |
| `turns` | int | User prompts so far |
| `elapsed_min` | double | Minutes since the session started |
| `model` | string | Current model as `provider/id` |
| `agent` | string | `main` or `sub` |

The design follows the classic **policy decision point / policy enforcement point** split. `decide()` is a pure function: it takes the rules, the facts, and the state, and returns a decision. It touches nothing, so it is easy to unit-test. The event handler is the enforcement point: it gathers the facts, calls `decide()`, and makes the switch.

## Four details that matter more than the rule language

**1. When you switch matters.** My first plan was to switch at `before_agent_start`, the hook that runs just before each turn. The agent's source showed a problem: a model change rebuilds that model's system prompt, and by `before_agent_start` the prompt for the turn is already built. The extension switches at `agent_end` instead. That is between prompts, the same moment a human would type `/model`.

**2. One-way ratchet.** Each rule fires at most once per session. A reversible rule ("go back to the big model when context is small") would invite switching back and forth, and each switch has a cost (see the next point).

**3. A switch is not free.** The new model starts with a **cold prompt cache**. The first prompt after the switch pays the full input price for the whole context. So a rule such as `tokens > 100000`, which switches *because* the context is big, can easily cost more than it saves. Rules based on spend, while the context is still small, are the safe kind. The tool says so in its notice: *"Next prompt starts with a cold cache."*

**4. The human wins.** If you change the model by hand after an automatic switch, the engine pauses for the rest of the session. The extension records its own switches as session entries, so this state is kept when you resume a session, and it can tell its own switch from yours.

## Verification

The unit tests cover rule compilation, first-match order, the ratchet, state replay after a resume, and cost accounting. The real proof is a live check: start the agent in RPC mode on model A, install the rule `turns >= 1 → B`, send two prompts, and check which model answered each one.

```
"assistantModels": ["<model-A>", "<model-B>"],
"notice": "[model-shift] Rule 'live-check' (turns >= 1) matched at $0.01, 6567 tokens: <A> → <B>. Next prompt starts with a cold cache.",
"passed": true
```

A second run also loads the decaying-effort extension. It confirms that a model switch is not mistaken for a manual effort change.

## What I would measure next

The open question is the break-even point: at what context size does the cold cache read cost more than the cheaper model saves? The extension logs every switch with the spend, tokens, and turns at that moment. A week of that log answers the question with data instead of guesses.

## Source

- [model-shift-omp](https://github.com/ankitg12/omp-extensions/tree/main/packages/model-shift-omp)
- [decaying-effort-omp](https://github.com/ankitg12/omp-extensions/tree/main/packages/decaying-effort-omp)
- [CEL specification and docs](https://cel.dev)
- [cel-js, the JavaScript CEL implementation used](https://github.com/marcbachmann/cel-js)
- [oh-my-pi coding agent](https://github.com/can1357/oh-my-pi)
