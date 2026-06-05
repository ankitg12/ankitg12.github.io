---
layout: post
title: "Model fit beats model rank"
date: 2026-06-05
categories: ai agents productivity
series: "AI coding agent productivity"
---

A model can be more capable and still be worse for the job in front of it.

That sounds obvious when stated abstractly. It is less obvious during daily use, when you switch an agent from one frontier model to another and expect a strictly better experience. Higher benchmark numbers create an intuition that the newer or stronger model should subsume the older one: more reasoning, more tool use, more context, more correctness.

A small morning experiment gave me a useful counterexample.

I ran the same personal-agent workflow in two different model configurations: one OpenAI-backed, one Claude Sonnet-backed. The job was not to write code. It was to produce a morning briefing: synthesize known context, identify what deserves attention, and recommend the next action.

The Sonnet-style run produced the right shape quickly: a concise briefing, a ranked set of options, one useful calendar observation, and a direct question about what to do next.

The OpenAI-style run was more diligent, but less useful. It read more context, checked more machinery, repeated some lookups, and drifted into implementation details. It was not wrong. It was miscalibrated.

That distinction matters.

The failure mode was not hallucination. It was not lack of intelligence. It was **mode mismatch**.

> Model quality is not scalar. It is model + mode + memory + tools + task shape.

If the mode is wrong, extra capability becomes drag.

---

## The task was not an engineering task

The user prompt was basically:

```text
Good morning.
```

In my setup, that is not small talk. It invokes a chief-of-staff agent with access to project notes, goals, calendar, queue, and recent activity. A good answer should do three things:

1. orient the user,
2. identify what matters,
3. recommend the next move.

It should not prove that it has searched every drawer in the house.

For a briefing task, the winning behaviour is not maximum retrieval. It is bounded judgment.

| Requirement | Good briefing behaviour | Bad briefing behaviour |
|---|---|---|
| Context | Load the few sources that change the decision | Re-read everything because it exists |
| Priority | Rank the options | Present a pile of facts |
| Observation | Surface one or two anomalies | Audit the system itself |
| Next step | Ask or recommend crisply | Continue gathering context |
| Tone | Calm, decisive, human | Procedural, over-verified, tool-centric |

The OpenAI-backed run behaved like a staff engineer entering a high-reliability codebase: verify everything, inspect sources, avoid unsupported claims, chase inconsistencies.

That is exactly what I want when changing production code.

It is not what I want when asking, "What should I pay attention to this morning?"

---

## Behavioural priors are product features

When people compare models, they often compress the question into one axis:

```text
Which model is smarter?
```

That framing hides the part users actually experience.

A model does not arrive as pure intelligence. It arrives with behavioural priors:

| Behavioural prior | Helps when | Hurts when |
|---|---|---|
| Verify aggressively | Code changes, security review, production debugging | Briefings, triage, executive summaries |
| Use tools often | Current facts matter | The answer is already in working memory |
| Explain thoroughly | Teaching, design docs, postmortems | Decision support |
| Preserve nuance | Architecture tradeoffs | Choosing the next action |
| Avoid risk | Regulated or safety-critical work | Lightweight personal workflow |
| Complete the whole task | Refactors, migrations, test repair | Conversational checkpoints |
| Generalize patterns | Framework design | One-off decisions |

None of these priors is universally good or bad. They are fit functions.

The same instinct that makes an agent excellent at a load-bearing refactor can make it irritating as a chief of staff. The same brevity that makes another model excellent at a briefing can make it under-eager during a deep debugging session.

This is why "has OpenAI caught up with Claude?" is not the right operational question.

A better question is:

> For this workflow, under this harness, with this memory and these tools, which model produces the behaviour I want?

---

## Benchmarks do not measure taste

Benchmarks are useful. They are also incomplete.

They can tell us whether a model solves math problems, writes code, uses tools, follows instructions, or passes agentic tasks in a controlled environment. They usually do not tell us whether the model has the right *taste* for a specific workflow.

For a personal chief-of-staff agent, the evaluation target is not just correctness. It includes:

- selectivity,
- timing,
- interruption cost,
- emotional calibration,
- ability to stop gathering context,
- willingness to make a recommendation,
- ability to distinguish user orientation from task execution.

Those are not soft concerns. They are product requirements.

An assistant that is always correct but always over-investigates has a high interaction tax. In enterprise environments, that tax is amplified: every extra lookup, every unnecessary qualification, every process detour competes with the user's actual work.

This is one reason generic "best model" selection often disappoints. Enterprise AI is not one workload. It is a portfolio of modes:

| Mode | Better default behaviour |
|---|---|
| Morning briefing | Synthesis, prioritisation, restraint |
| Engineering execution | Verification, tests, source grounding |
| Incident response | Speed + evidence + escalation discipline |
| Coaching | Challenge, reflection, leverage |
| Writing | Structure, voice, revision taste |
| Research | Source quality, coverage, uncertainty handling |

A single leaderboard score cannot choose across those modes.

---

## The useful split: briefing model vs engineering model

After this experiment, I would not say "use model A" or "use model B" globally.

I would split the operating modes.

### Briefing mode

Use the model or prompt shape that behaves like a chief of staff:

```text
Use only the minimal sources needed for the decision.
Return a ranked briefing.
Surface at most one anomaly.
End with one recommended next action.
Do not inspect implementation details unless explicitly asked.
```

Success criteria:

- The user knows what matters.
- The user has fewer open loops.
- The agent does not turn orientation into archaeology.

### Engineering mode

Use the model or prompt shape that behaves like a staff engineer:

```text
Ground claims in repository or runtime evidence.
Inspect call sites before changing exported APIs.
Update affected tests.
Run the narrowest meaningful verification.
Do not yield with unverified behavioural changes.
```

Success criteria:

- The change works.
- The tests cover what can break.
- The next maintainer can understand it.
- The agent does not substitute a plausible patch for the real fix.

### Coach mode

Use a hybrid:

```text
Ask what outcome the user wants.
Challenge low-leverage work once.
Connect the task to long-term goals.
Then help execute.
```

Success criteria:

- The user makes a better decision.
- The agent does not create busywork.
- The conversation stays human.

This is not model routing by brand. It is model routing by behavioural contract.

---

## What an eval for this would look like

The right way to compare models for this workflow is not a vibe check. It is a small eval suite built from real usage.

For a morning briefing agent, I would start with 20-50 transcript-derived tasks. Each task would include a controlled bundle of inputs: goal file, latest note, project list, calendar summary, queue, and recent activity. The model output would be graded on outcomes, not internal steps.

Example grader dimensions:

| Dimension | Check |
|---|---|
| Priority selection | Did it identify the highest-leverage next action? |
| Source restraint | Did it avoid unnecessary context gathering? |
| Anomaly detection | Did it catch the one calendar/process inconsistency? |
| Human calibration | Did it ask how the user is when appropriate? |
| Actionability | Did it end with a clear recommendation or choice? |
| Privacy | Did it avoid leaking sensitive personal or workplace details? |
| Verbosity | Was it concise enough to be read before work begins? |

Some of these can be code-graded: output length, presence of a recommendation, absence of sensitive strings, number of tool calls.

Some need rubric grading: judgment quality, tone, prioritisation.

The important point is that the eval should measure the product requirement, not the model's general intelligence.

OpenAI's own eval guidance says to design task-specific evals that reflect real-world distributions, avoid vibe-based evaluation, and combine metrics with human judgment. Anthropic's agent eval writing makes a similar point: evaluate the agent harness and model together, using transcripts and outcome checks, because agent behaviour compounds over turns.

That is exactly the lesson here. The thing being evaluated is not "OpenAI" or "Claude" in isolation. It is:

```text
model + agent harness + instructions + memory + tools + task
```

Change any one of those and the result can change.

---

## The enterprise lesson

This is where the comparison becomes more general.

Enterprise AI buyers and builders often ask:

```text
Which model should we standardize on?
```

A more useful architecture question is:

```text
Which behaviours do our workflows require, and how do we route or shape models to produce them reliably?
```

For enterprise AI, the hard part is rarely raw capability alone. The hard part is behavioural fit under constraints:

- Does the model know when to stop?
- Does it ask before acting, or act before asking?
- Does it overuse tools?
- Does it under-verify risky changes?
- Does it preserve confidentiality by default?
- Does it produce outputs that match the user's time horizon?
- Does it degrade gracefully when context is stale?

A model that is excellent for code repair may be too procedural for executive assistance. A model that writes elegant summaries may miss source-level edge cases in a refactor. A model that is cautious in a regulated workflow may feel unusably slow in personal productivity.

So "catching up" is not a single event.

A lab can catch up on benchmarks and still differ in behavioural priors. A model can win a coding eval and lose a morning briefing. Another can be delightful in conversation and insufficiently rigorous in a safety-critical change.

The practical enterprise stack should therefore look less like this:

```text
one model to rule them all
```

and more like this:

```text
workflow → behavioural contract → eval suite → model/prompt/tool routing
```

---

## My current rule

For now, my rule is simple:

| Situation | Prefer |
|---|---|
| Brief me | The model configuration with restraint and prioritisation |
| Change code | The model configuration with verification and tests |
| Review a plan | A second model as critic |
| Decide next action | The model that can stop gathering context |
| Investigate production risk | The model that refuses to hand-wave |

The best agent is not the one that is always most powerful.

The best agent is the one whose failure mode matches the cost of being wrong.

For a morning briefing, the cost of being wrong is misdirected attention. The agent should be selective.

For engineering work, the cost of being wrong is a shipped bug. The agent should verify.

Those are different jobs.

Treating them as the same job is how a very capable model becomes a very well-dressed nuisance.

---

## Sources

- Anthropic, [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- OpenAI, [Evaluation best practices](https://developers.openai.com/api/docs/guides/evaluation-best-practices)
- Arize, [The definitive guide to LLM evaluation](https://arize.com/llm-evaluation/)

_Written by GPT-5.5._
