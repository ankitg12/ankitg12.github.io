---
layout: post
title: "Before the Agent Acts: Designing a Prompt-Intake Layer for Coding Agents"
date: 2026-07-27
categories: ai agents productivity engineering
series: "AI coding agent productivity"
---

A coding agent can execute a bad interpretation with astonishing efficiency.

The usual response is to tell users to write better prompts. That is useful advice, but it puts the quality boundary in the wrong place. A harness already controls the moment between **raw intent** and **an expensive agent turn**. It can inspect the request, pass clear work through, and stop ambiguous work long enough to ask the questions that materially change the result.

I wanted this flow:

```text
raw request
  -> intake analysis
  -> questions only if needed
  -> reviewed execution brief
  -> fresh main agent
```

Before building it, I searched OMP/Pi and adjacent coding-agent ecosystems for an existing implementation. The search found almost every necessary piece—but not one mature package that combines them correctly.

This post records that search, including the attractive options that turned out not to fit.

## Intake is not planning

The first design mistake is to treat intake and planning as synonyms.

An **intake layer** answers:

- Is there a clear outcome?
- Is the deliverable identifiable?
- Which missing facts would materially change the work?
- Are several goals competing?
- Can the request pass through unchanged?

A **planner** answers:

- Which files and symbols are involved?
- What implementation sequence should be followed?
- Which edge cases and tests belong in the execution spec?

Planning is downstream. If the harness sends “make authentication better” directly to a planner, the planner may produce a very thorough plan for an interpretation the user never intended.

A useful intake system therefore needs six properties:

| Property | Why it matters |
|---|---|
| Fast path for clear prompts | Better prompting must not make every trivial task slower |
| Selective questions | Every question must change the output or close a load-bearing gap |
| Explicit current hypothesis | The user needs to see what the system currently thinks they mean |
| Bounded dialogue | Intake cannot become an endless requirements workshop |
| Reviewed canonical brief | Execution needs a stable contract, not an improvised summary |
| Fresh execution context | The main agent should not inherit abandoned hypotheses and interview debris |

That last property is easy to miss. A refined prompt is not enough if the executor also receives the entire conversation that produced it. Early guesses remain in context and can compete with the final decision.

## The DRW search

I evaluated candidates using a “don't reinvent the wheel” filter, but not a stars-only filter. The preference order was:

1. established, human-built machinery with evidence of sustained use;
2. newer projects with real multi-commit history, tests, and behavioral evaluation;
3. tiny or brand-new projects only as idea sources—not production dependencies.

Stars below are point-in-time observations from 27 July 2026. They indicate adoption, not correctness.

## OMP `/guided-goal`: the engine is already there

[OMP](https://github.com/can1357/oh-my-pi) already contains the closest native primitive: `/guided-goal`.

Its implementation in [`guided-setup.ts`](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/goals/guided-setup.ts) does several things that a good intake layer needs:

- selects the `plan` model role, then `slow`, then the current model;
- forces a structured `question | ready` response through a tool schema;
- routes the interview through a side-session ID distinct from the main session;
- reuses that side session across turns;
- applies the same secret obfuscation used by normal turns;
- returns a structured objective instead of free-form prose.

The interactive handler then asks one question per turn, caps the interview at six turns, opens a final review editor, and starts Goal Mode with the approved objective.

This is not just similar to intake. It is most of the hard session machinery.

The mismatch is at the handoff boundary: `/guided-goal` can only start autonomous Goal Mode, and its schema carries an `objective`, not a normal-task execution brief. The right move is to generalize this engine, not write a second interview loop.

**Maturity:** OMP had roughly 19.9k stars and 15,640 commits when checked. More importantly, this machinery is already on the execution path we need to integrate with.

## OMP Plan Mode: promising documentation, wrong observed behavior

OMP Plan Mode looks like an obvious answer. It is read-only, may interview the user, writes a self-contained execution spec, and lets the user approve execution in a fresh context.

Its own contract says:

```text
large or unspecified task -> multiple interview rounds
small or well-specified task -> few or no questions
```

I tested that assumption in three disposable `omp --no-session` instances.

| Input | Observed behavior |
|---|---|
| A clear, tightly scoped command-description change with an explicit success condition | Repository exploration followed by a 28-line plan |
| “Make `/guided-goal` work better for normal coding tasks” | Several planning scouts launched before asking what “better” meant |
| A request with three related goals | Architectural investigation and a provisional plan before priorities were clarified |

The result was consistent: Plan Mode behaved as a **grounded execution planner**, not a lightweight intake gate. That is not a Plan Mode defect. It is doing the job it was built to do.

Turning on `plan.defaultOnStartup` would therefore move the boundary but not solve the problem. Every prompt would pay planning overhead, and ambiguous intent could still survive into a detailed plan.

## Prompt Preflight: the best deterministic front gate

[Prompt Preflight](https://github.com/akg268/prompt-preflight) describes itself as a local prompt-quality layer that runs before coding agents and AI workflows.

At the time of review it had only three stars, but the repository was substantially healthier than that number suggested:

- 103 commits;
- unit tests and a fixed benchmark of intentionally vague prompts;
- Codex, Claude Code, and Kiro hook adapters;
- block and nudge modes;
- no model call, API key, network access, or prompt upload;
- structured JSON output for evaluation and integration.

Its analyzer looks for more than “short prompt equals bad prompt.” It checks categories such as:

- clarity and subjective wording;
- missing context, files, data, or attachments;
- missing output contract or success criteria;
- production, migration, destructive, or broad-scope risk;
- work that should start with a plan;
- likely secrets in the prompt.

This is the strongest candidate for the **cheap first gate**. Clear work can pass without an LLM call; obviously underspecified or high-impact work can be intercepted before repository exploration begins.

Its limitation is equally important: it produces a stronger prompt and targeted questions, but it is not an isolated multi-turn interview session. It also has no OMP adapter. I would borrow its signals and evaluation discipline, not install it wholesale into OMP.

## PRR: almost the exact terminal UX

[PRR—Prompt Refine & Run](https://github.com/ravistakumar/prr) is the closest end-to-end match to the desired user experience:

```text
prr "add dark mode"
  -> detect lightweight repository signals
  -> ask a headless agent to optimize and score confidence
  -> confidence low? ask 1-3 terminal questions
  -> hand the refined prompt to the coding agent
```

It supports Claude Code, Codex, OpenCode, and Aider, and it can print the refined prompt instead of launching an agent. Its fail-open behavior returns to the original prompt if refinement fails.

Architecturally, this is clean: refinement happens before the main process starts, so context separation comes naturally.

But PRR had zero stars and no OMP adapter when reviewed. It was too new to become the foundation of a load-bearing OMP workflow. Its value is as a compact reference architecture: **confidence gate, bounded questions, confirmed handoff**.

## Agentic Prompt Intake: the best routing vocabulary

[Agentic Prompt Intake](https://github.com/vetlucasmartins/agentic-prompt-intake) defines exactly the conceptual pipeline:

```text
raw user input -> intake router -> intake refiner -> main executor
```

Its four labels are useful:

- `READY_TO_EXECUTE`
- `NEEDS_LIGHT_REFINEMENT`
- `NEEDS_INTAKE`
- `BLOCKED`

The protocol also includes activation and suppression signals, readiness and ambiguity scores, JSON schemas, templates, examples, and an evaluation runner that measures classification accuracy, question count, over-triggering, under-triggering, and token ceilings.

That evaluation framing is more valuable than the prompt text. Intake systems fail in two directions:

- **under-triggering:** the agent executes a consequential ambiguity;
- **over-triggering:** the harness interrogates the user about routine work that a competent default would solve.

The project had one star when checked and primarily packages instructions and skills. That means enforcement depends on model compliance; it does not create a hard context boundary. Again, the correct use is selective adoption: take the taxonomy and evaluation cases, not the whole mechanism.

## Addy Osmani's `interview-me`: the best conversational policy

The [`interview-me`](https://github.com/addyosmani/agent-skills/blob/main/skills/interview-me/SKILL.md) skill in Addy Osmani's [agent-skills](https://github.com/addyosmani/agent-skills) repository supplies the most useful interview rule:

> Ask one question, then state the current hypothesis.

That pairing is better than a list of generic questions. It exposes the model's interpretation while it is still cheap to correct. The user can answer the question and repair the hypothesis in one turn.

The repository had roughly 80.5k stars when checked, making it the most widely adopted source in this comparison. But a skill is still an instruction. It does not, by itself, guarantee that the intake conversation is isolated from execution or that a fresh agent receives only the final brief.

The policy belongs inside the OMP side-session engine.

## Pi Prompt Refiner and Pi Agent Suite

[Pi Prompt Refiner](https://github.com/basuev/pi-prompt-refiner) performs a one-shot refinement before sending the result onward. It demonstrates that Pi-family extensions can reshape user input, but it does not run an interview or create a separate executor context. The repository had one star when reviewed.

[Pi Agent Suite's structured prompt extension](https://github.com/n-r-w/pi-agent-suite/blob/main/docs/extensions/structured-prompt.md) provides a fixed form for fields such as goal, context, constraints, and expected output. This is useful when the user already knows how to specify the work. It does not discover missing intent conversationally.

Both help with prompt shape. Neither supplies the complete intake boundary.

## Superpowers: strong design discipline, too much policy for every prompt

[Superpowers](https://github.com/obra/superpowers) includes a brainstorming workflow that prevents implementation before a design is understood and approved. It asks questions, presents a design in reviewable sections, writes a design document, and moves into implementation planning.

That discipline is valuable for features and architecture. It is too heavy as a universal first-prompt gate. Making every ordinary request pass through mandatory brainstorming, documentation, and planning would trade wrong-direction execution for process tax.

The reusable lesson is narrower: **do not implement while the user and agent disagree about the problem**.

## Spec Kit, OpenSpec, and GSD: durable specification workflows

Three larger frameworks solve adjacent problems:

- [GitHub Spec Kit](https://github.com/github/spec-kit) uses `specify -> clarify -> plan -> tasks -> implement`. Its ambiguity taxonomy and explicit clarification phase are useful, but it is optimized for specification-driven project work.
- [OpenSpec](https://github.com/Fission-AI/OpenSpec) separates exploratory conversation from a formal proposal. It is lighter than Spec Kit, but still centered on durable project artifacts rather than first-turn routing.
- [GSD](https://github.com/open-gsd/gsd-core) has a strong discuss-phase and durable `CONTEXT.md` handoff. It is a full project-delivery workflow, not a prompt firewall.

These frameworks are good downstream destinations. A prompt-intake layer should be able to decide that a request deserves one of them. It should not become one of them.

## Cline and Aider plan modes

Cline and Aider both support plan/read-only styles of work before editing. They reinforce the same distinction exposed by the OMP trial: a plan mode controls **how the agent approaches implementation**. It does not necessarily decide whether the user's intended outcome is known well enough to plan.

The existence of a planner does not remove the need for intake.

## The architecture that survives the comparison

The search converged on a small composition rather than a new framework:

```text
first ordinary prompt
  |
  v
cheap readiness gate
  |-- READY ------------> original prompt -> fresh main agent
  |-- BLOCKED ----------> explain missing prerequisite; stop
  `-- NEEDS_REFINEMENT
          |
          v
   isolated intake side session
   - one question at a time
   - current hypothesis visible
   - default 3, hard maximum 6
          |
          v
   user reviews execution brief
          |
          v
   original prompt + final brief
          |
          v
   fresh main agent
```

Each part has prior art:

| Component | Source to reuse |
|---|---|
| Side-session model routing, secret handling, bounded turns | OMP `/guided-goal` |
| Cheap activation and suppression signals | Prompt Preflight |
| `READY / REFINE / BLOCKED` routing contract | Agentic Prompt Intake |
| One question plus current hypothesis | `interview-me` |
| Confidence-gated terminal handoff | PRR |
| Self-contained downstream execution contract | OMP Plan Mode and Spec Kit |

The canonical brief should carry:

```text
Original request
Refined objective
Deliverables
Constraints and non-goals
Success criteria
Relevant context
Confirmed decisions
Remaining explicit assumptions
```

Keeping the original request is deliberate. Prompt optimization is a lossy transformation. A polished brief can accidentally remove tone, priority, uncertainty, or a seemingly incidental constraint. The executor should receive both the source and the reviewed interpretation.

The intake transcript should not cross the boundary. Its purpose is to produce the contract, not become part of it.

## How I would integrate it into OMP

The clean implementation path is to generalize `/guided-goal` into target-neutral intake machinery while retaining `/guided-goal` as an adapter.

Two entry points are useful:

```text
/intake <prompt>       explicit, deterministic invocation
firstPrompt: auto      optional automatic gate
```

A conservative configuration might look like:

```yaml
intake:
  firstPrompt: off      # off | suggest | auto
  maxQuestions: 3       # hard ceiling remains 6
```

`off` preserves current behavior. `suggest` offers intake when the gate finds material ambiguity. `auto` enters the interview automatically but still passes clear prompts through unchanged.

This belongs in the OMP session/mode boundary rather than a prompt-only skill. An ordinary extension can reshape text, but cleanly replacing an intake-only conversation with a fresh interactive executor requires lifecycle control. The context boundary is the feature.

## What not to build

The comparison rules out several tempting designs:

- **Do not enable Plan Mode for every session.** Planning overhead is not intake.
- **Do not rewrite every prompt.** Clear prompts should remain byte-for-byte recognizable.
- **Do not ask for optional preferences.** Reasonable defaults are part of agent competence.
- **Do not carry the interview transcript into execution.** It preserves rejected hypotheses.
- **Do not use prompt length as ambiguity.** A short prompt can be exact; a long voice transcript can remain unclear.
- **Do not make intake another mandatory spec framework.** It should route into planning when planning is warranted.
- **Do not trust a plausible prompt policy without behavioral evaluations.** Over-triggering is a product defect.

## A minimal evaluation set

The live Plan Mode trial suggests a small regression suite for any intake implementation:

| Case | Input shape | Expected result |
|---|---|---|
| Clear | Named change, target, constraint, success condition | Pass through; no questions |
| Vague | “Make X better” | Ask which observable outcome matters |
| Multi-goal | Several outcomes without priority | Surface priorities or encode explicit constraints |
| Missing prerequisite | References an absent attachment or credential | Block without execution |
| Voice narration | Self-correction and several possible deliverables | Synthesize understanding, then ask focused questions |
| Low-risk omission | Optional style or naming preference only | Use a default; do not trigger full intake |

The critical metrics are not “prompt quality score” or output length. They are:

- false-positive intake rate;
- false-negative ambiguity rate;
- average questions per triggered request;
- user edits to the final brief;
- whether the executor receives rejected assumptions;
- whether the completed work matches the reviewed outcome.

## The broader lesson

Agent harnesses usually invest heavily in execution controls: permissions, sandboxes, planning modes, tool schemas, checkpoints, and tests. The input boundary receives much less engineering attention.

That boundary has unusually high leverage. A five-second clarification can prevent a twenty-minute repository investigation, a detailed wrong plan, and an implementation that must be unwound. But the solution is not to interrogate users before every turn. It is to distinguish **uncertainty that changes the result** from **details a competent agent should resolve itself**.

The DRW search did not find a package to install and forget. It found something better: a mature native engine, a deterministic gate, a routing vocabulary, a conversational policy, and an evaluation method that fit together without importing an entire methodology.

Sometimes “don't reinvent the wheel” means installing the wheel. Sometimes it means noticing that five projects have already built the spokes—and that your own harness already contains the hub.

## Sources

- [Oh My Pi](https://github.com/can1357/oh-my-pi)
- [OMP guided-goal implementation](https://github.com/can1357/oh-my-pi/blob/main/packages/coding-agent/src/goals/guided-setup.ts)
- [Prompt Preflight](https://github.com/akg268/prompt-preflight)
- [PRR — Prompt Refine & Run](https://github.com/ravistakumar/prr)
- [Agentic Prompt Intake Protocol](https://github.com/vetlucasmartins/agentic-prompt-intake)
- [Addy Osmani's agent-skills](https://github.com/addyosmani/agent-skills)
- [`interview-me`](https://github.com/addyosmani/agent-skills/blob/main/skills/interview-me/SKILL.md)
- [Pi Prompt Refiner](https://github.com/basuev/pi-prompt-refiner)
- [Pi Agent Suite structured prompt](https://github.com/n-r-w/pi-agent-suite/blob/main/docs/extensions/structured-prompt.md)
- [Superpowers brainstorming](https://github.com/obra/superpowers/tree/main/skills/brainstorming)
- [GitHub Spec Kit](https://github.com/github/spec-kit)
- [OpenSpec](https://github.com/Fission-AI/OpenSpec)
- [GSD](https://github.com/open-gsd/gsd-core)
- [Requirements smells as ambiguity indicators](https://arxiv.org/abs/2404.11106)
