---
layout: post
title: "A Session Governor: One Policy Engine for the Whole Agent Session"
date: 2026-10-06 20:30:00 +0530
categories: ai agents productivity
series: "AI coding agent productivity"
---

Yesterday a cheap model spent an hour failing to diagnose an SSH outage. It rejected the correct cause when I gave it to it, and four minutes after I switched to a stronger model by hand, the problem was solved.

Nothing in the session could see that the agent was stuck. A budget rule had put the cheap model there, and no rule knew how to take it away.

This week I wrote three small extensions for the [oh-my-pi](https://github.com/can1357/oh-my-pi) coding agent. Each one changed one setting of a running session:

| Extension | Setting | Post |
|---|---|---|
| decaying-effort | Thinking effort, high at the start, lower later | [Decaying Effort]({% post_url 2026-10-05-decaying-effort-transmission-gearbox-ai-agents %}) |
| model-shift | Which model, chosen by CEL rules over cost and turns | [Policy-as-Code]({% post_url 2026-10-06-policy-as-code-for-your-coding-agents-wallet %}) |
| model-switch-prune | What context is sent, by trimming old tool output | [Lighter Sessions]({% post_url 2026-10-06-lighter-sessions-on-model-switch-wire-only-context-pruning %}) |

Each one worked on its own. Together they fought. This post is about the merge, and about the fourth setting the merge made easy to add: escalation when the agent says it is stuck.

## Why three controllers is the wrong number

Three controllers that act on the same session at different times cause three problems:

1. **Each one invalidates the prompt cache on its own.** A model switch makes the cache cold. A prune changes the prompt prefix, so the cache is cold again. When the two happen on different turns, you pay for two cold reads instead of one.
2. **Each one has to tell the others' changes from yours.** The effort controller needed special code so that a model switch did not look like a manual effort change. The model controller treated a `/model` as "the human took over" and stopped all of its rules, including the effort part of each rule.
3. **No one owns the policy.** Three config files, three sets of rules, and no single place that answers "why is this session on this model at this effort right now?"

Control engineers know this pattern. Put several independent loops on one plant and they interact: one loop's output is another loop's disturbance. The usual fix is one supervisor that sees every sensor and owns every actuator.

## The governor

`session-governor-omp` is that supervisor. It has one rules file, one evaluation point, and one log.

```yaml
rules:                     # ordered; first match wins
  - name: away
    when: 'afk'
    use: '@smol'
    revert: true           # restore model and effort when I am back

  - name: stuck-escalate
    when: '(blocked_streak >= 3 || attempts_on_goal >= 6) && !model.startsWith("big/")'
    use: '@slow'
    effort: high

  - name: spend-cap
    when: 'cost >= 1.0 && model.startsWith("big/") && blocked_streak == 0'
    use: '@mid'

  - name: context-epoch
    when: 'tokens > 80000 && turns_since_prune >= 4'
    prune: true
    repeat: true

  - name: effort-high
    when: 'turns >= 2 && cost < 1.0'
    effort: high
  - name: effort-auto
    when: 'turns >= 4'
    effort: auto
```

The structure is the same as in the policy-as-code post, but the scope is the whole session:

| Part | What it is |
|---|---|
| **Sensors** | CEL variables: `cost`, `tokens`, `context_pct`, `turns`, `elapsed_min`, `model`, `agent`, `afk`, `turns_since_prune`, `blocked_streak`, `attempts_on_goal` |
| **Policy** | An ordered list of [CEL](https://cel.dev) rules. CEL is not Turing-complete, has no side effects, and is type-checked when the config loads |
| **Decision** | `decide(rules, vars, state)` is a pure function, so unit tests need no agent |
| **Actuators** | `use:` (model), `effort:`, `prune:`; `revert:` and `repeat:` change how a rule re-arms |
| **Evaluation point** | `agent_end`, between two prompts, the same moment a human would type `/model` |

Because there is one decision per boundary, a rule that switches the model and prunes in the same step costs **one** cold cache read.

## Who wins: the human

A governor that fights the human is worse than none. The rule is simple: the human wins, but only on the setting the human touched.

| You change | Paused | Still active |
|---|---|---|
| The model (`/model`) after a governor switch | Rules with `use:` | Effort and prune rules |
| The effort after a governor effort change | Effort-only rules | Model and prune rules |

Both pauses are written into the session as custom entries, so they survive a resume. The governor tells its own changes from yours by comparing with what it last applied. For effort, it compares the *selector* the user picked, not the resolved level, because `auto` resolves again on every prompt.

## The fourth sensor: let the agent say it is stuck

The missing sensor was the one I needed yesterday. Cost and turns are easy to measure. "This agent is going nowhere" is not.

I looked at three ways to sense it:

| Approach | Problem |
|---|---|
| Watch the user (repeated corrections, frustration words) | Slow, and it needs the human to be annoyed first |
| A second model as a judge | Costs money every turn, and is one more model to trust |
| **The agent reports its own progress** | Cheap. Is it honest? |

I chose self-report. The governor registers a small tool, `progress`, that is always loaded:

```json
{ "goal": "find why ssh to the lab host fails",
  "status": "blocked",
  "evidence": "Permission denied (publickey) again after key reload" }
```

The agent calls it once per turn. The governor reads those calls back from the session branch and computes two variables:

- `blocked_streak`: how many `blocked` reports in a row on the current goal.
- `attempts_on_goal`: how many reports of any status since the last `done`.

The second counter matters more. Yesterday's model never said "blocked". It said "progress" again and again while it made none. Six reports on one goal without a `done` is a signal, whatever the status says. A new goal text resets both counters, so normal chat does not count against the agent.

Because the counters are derived from the branch and not kept in memory, they are correct after a resume and they follow you when you move to another branch.

A live check ran a real agent in RPC mode. Two different models called the tool on their own, without any prompt asking them to:

```
"progressCalls": [
  {"goal": "Reply with the single word: one", "status": "done", "evidence": "one"},
  {"goal": "Reply with the single word: two", "status": "done", "evidence": "two"}
],
"passed": true
```

## What is not proven yet

- **Honesty under failure.** The live check shows that models *call* the tool. It does not show that a weak model says `blocked` when it is stuck. The next experiment gives a small model a task it cannot do and counts its reports.
- **Goal rewording.** A stuck model can reset the counters by changing its goal text. Normalizing case and spaces does not catch a real rewrite.
- **Cost.** Each turn has one more tool call.
- **Silence.** A model that never calls the tool never escalates.

## Related work

I am not the first to route an agent between models. [pi-model-router](https://github.com/yeliu84/pi-model-router) chooses a high, medium or low tier for each turn from task intent, budget, context size and keyword rules, and sets the thinking level for each tier. It is a good answer to the question "which model for this prompt?".

The governor asks a different question: "what should this *session* look like now?". The differences:

- One general expression language over session facts, not a fixed set of heuristics.
- Context pruning is an action next to model and effort, so their cache costs are paid together.
- Separate manual-override pauses for each setting.
- A progress signal from the agent itself as an input.

I did not find another tool that combines these four. That is a weak claim, and I would like to hear about prior art.

## Source

- [session-governor-omp](https://github.com/ankitg12/omp-extensions/tree/main/packages/session-governor-omp)
- [CEL specification and docs](https://cel.dev)
- [cel-js, the JavaScript CEL implementation used](https://github.com/marcbachmann/cel-js)
- [oh-my-pi coding agent](https://github.com/can1357/oh-my-pi)
- [pi-model-router](https://github.com/yeliu84/pi-model-router)
