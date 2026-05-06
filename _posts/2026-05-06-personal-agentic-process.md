---
layout: post
title: "Personal Agentic Process — engineering discipline for human-agent pairs"
date: 2026-05-06
categories: ai agents productivity engineering
series: "AI coding agent productivity"
---

In 1995, Watts Humphrey published the Personal Software Process (PSP): a disciplined framework for individual software engineers to measure and improve their own performance. Time logging per phase. Defect tracking from injection to removal. Size estimation using historical data. Post-mortems. Checklists. The unit of analysis was the individual developer, and the insight was that *you can't improve what you don't measure*.

Thirty years later, we have AI coding agents that write most of the code. The unit of analysis has shifted. You are no longer the process — the **human-agent pair** is the process. PSP's discipline still applies, but the measurements change, the instrumentation changes, and several problems PSP couldn't solve are now solvable automatically.

This post proposes **PAP — Personal Agentic Process** — a lightweight discipline for human-agent pairs, drawing on PSP's structure, adapted for the tooling available in 2026.

---

## What PSP got right

PSP's core insight: improvement requires data, and data requires discipline. The four things PSP measured:

- **Time** — how long each phase took vs. estimated
- **Defects** — where they were injected, where they were found, how long removal took
- **Size** — lines of code, used for estimation calibration
- **Process** — which phases produced which defects

From these four measures, PSP derived yield (% defects removed before test), cost of quality, and estimation accuracy. Over time, a developer built a personal baseline and could predict their own performance.

The problem: all of this was *manual*. Developers logged time in spreadsheets. They categorized defects by type. The overhead was real enough that PSP adoption was low outside of formal training programs.

---

## What changes with agentic development

Three things shift fundamentally:

**1. The agent generates most artifacts, not the human.**
The human's primary output is *judgment* — approving plans, redirecting stuck agents, reviewing output, deciding when to push and when to pull back. Code volume is no longer a useful size metric. Tool calls and segment activity are more meaningful.

**2. The agent can be instrumented automatically.**
Every tool call, every turn, every elapsed second can be captured without manual effort. The overhead that killed PSP adoption doesn't exist in a properly instrumented session.

**3. The agent drifts in ways humans don't.**
A human developer doesn't forget the goal mid-task. An agent will optimize locally, go in circles, over-engineer, or silently drop context after compaction. Drift detection requires *segment-level* reflection — not just a final post-mortem.

---

## The PAP framework

PAP has four components, mapped against their PSP equivalents:

| PSP | PAP |
|---|---|
| Time logging (manual) | Automatic timestamps on every message and turn |
| Defect tracking | Backtrack detection in segment retros ("Stuck?" question) |
| Size measurement | Tool call count and type per segment |
| Post-mortem | Clock-aligned segment retrospection (every 10 min / 60 min) |
| Process improvement proposals | Session knowledge capture and publication |
| PROBE estimation | Session goal-setting before starting |
| Phase checklists | Structured retro prompt applied every segment |

### Component 1: Temporal instrumentation

The agent and the human operate on different time scales. The agent has no clock. The human loses track of elapsed time. Fixing both requires injecting time into the conversation itself.

Two mechanisms, built as OMP extensions:

**Input timestamps** — every user message is prepended with `[HH:MM:SS]` via an `input` event transform. Deterministic, no LLM involvement. The transcript becomes a timeline.

**System prompt injection** — before each agent turn, the current time is injected into the system prompt with an instruction to prepend it to responses. Best-effort, but consistent enough to anchor the conversation.

**Status bar** — live `[04:42:45 PM] bash` during tool execution, `[04:43:12 PM] 27s` after each turn. Elapsed time visible without reading the transcript.

The result: every segment of the conversation has a timestamp. You can see when things started, how long they took, and where time was spent.

See: [Adding timestamps to AI coding agent conversations]({% post_url 2026-04-27-timestamps-for-ai-coding-agents %})

### Component 2: Clock-aligned segment retrospection

PSP's post-mortem happened once, at the end of a task. By then, the detail was gone. PAP fires retrospection every 10 minutes, while the evidence is still in the session.

The mechanism: a clock-aligned OMP extension fires a structured retro prompt at wall-clock boundaries (:00, :10, :20...). The agent receives the prompt as a follow-up message, with an activity summary prepended:

```
Segment: 04:10 PM → 04:20 PM (10min), 31 calls (18 bash, 8 edit, 3 read, 2 web_search)
```

The retro prompt asks eight questions, answered separately:

1. **Done** — was the goal achieved? right tradeoffs?
2. **Stuck?** — any repeated attempts or backtracking?
3. **Don't reinvent the wheel** — did you check for existing tools first?
4. **Knowledge expansion** — what adjacent tool or technique is worth knowing?
5. **Artifacts** — what should be published for others?
6. **Documented?** — will a new session re-discover this?
7. **One thing at a time** — scatter or focus?
8. **Engineering principles** — YAGNI, DRY, KISS, SRP, PoLA, PoLP violations?

If no tool calls occurred in the segment, the retro is skipped silently. The retro fires only when there's evidence to reflect on.

The hourly retro uses a longer prompt — a full session review, not just the last 10 minutes.

See: [Clock-aligned retrospection for AI coding agents]({% post_url 2026-05-06-clock-aligned-retro-for-ai-agents %})

### Component 3: Session discipline

The existing agentic engineering literature (Tyler Burleigh's RPIR, avanwyk's 8-phase workflow, Spec-Driven Development) converges on one insight: **unstructured sessions produce unreliable results**. What they don't address is the *within-session* discipline — how to handle the 90 minutes between planning and commit.

PAP's session structure:

**Session start:**
- State the goal in the first message (the agent reads it; timestamps anchor when you started)
- Seed context explicitly — relevant files, prior session knowledge, constraints
- Commit or stash anything uncommitted before starting

**During the session:**
- Let retros fire — don't dismiss them, don't work through them
- When the retro says "stuck" and you're on the third variant of a failing approach, stop
- When the retro says "reinvented the wheel", check if the code should be a shared abstraction

**Session close:**
- Update knowledge files with anything a new session shouldn't have to re-discover
- Commit with a message that captures the *why*, not just the *what*
- Note blockers explicitly rather than leaving partial work unmarked

### Component 4: Knowledge capture as process output

PSP measured individual performance but didn't produce artifacts for others. PAP treats knowledge capture as a first-class output — equal in value to code.

Every retro asks "will a new session re-discover what was learned?" This isn't just documentation hygiene — it's the long-term memory of the process. Knowledge files persist across sessions. They inform the agent's system prompt. They reduce the cost of every future session that touches the same domain.

The discipline: knowledge captured in a retro must be *committed and pushed* before the session ends. Written but uncommitted knowledge is Schrödinger's documentation — it may or may not exist next session.

---

## PAP vs existing agentic workflows

The research-plan-implement-review workflows (RPIR, SDD, RPI) address macro-level structure: what phases to run, how to manage context windows, when to start fresh sessions. They're correct and worth following.

PAP operates at a different level — *within* sessions, not *across* them. The two complement rather than compete:

| | RPIR/SDD/RPI | PAP |
|---|---|---|
| Unit | Task / story | Segment (10 min) |
| Focus | Phase transitions | Within-session drift |
| Measurement | Manual or absent | Automatic (tool calls, time) |
| Reflection | End of phase | Every 10 minutes |
| Memory | PLAN.md, RESEARCH.md | Knowledge files + retro log |

A full workflow: use RPIR for task structure, use PAP for session discipline within each phase.

---

## Implementation on OMP

Three OMP extensions, two config entries:

**`chat-timestamps`** — temporal instrumentation
```yaml
# ~/.omp/agent/config.yml
extensions:
  - ~/repos/github.com/ankitg12/chat-timestamps
```

**`agent-retro-omp`** — clock-aligned retrospection
```yaml
extensions:
  - ~/repos/github.com/ankitg12/agent-retro-omp
```

```json
// ~/.omp/agent/agent-retro.json
{
  "retros": [
    { "name": "mini",   "intervalMs": 600000,  "prompt": "~/.claude/commands/retro.md" },
    { "name": "hourly", "intervalMs": 3600000, "prompt": "~/.claude/commands/retro-long.md" }
  ]
}
```

After restarting OMP: `~/.omp/agent/agent-retro.log` should show `session_start — arming 2 retro(s)`.

---

## What PAP can't do yet

PSP eventually produced *calibrated estimation* — using historical defect and time data to predict future project cost. PAP doesn't have this yet. The retro log captures segment activity, but there's no tooling to aggregate it across sessions into a personal baseline.

The data is there. Session files are JSONL. The retro log records segment call counts. A session analysis script could produce: average tool calls per segment, most common "stuck" patterns, which project types cause the most backtracking, how often knowledge capture actually happens. That's a future piece of tooling — the equivalent of PSP's Process Dashboard.

---

## Sources

- Watts Humphrey, *A Discipline for Software Engineering* (1995) — PSP original
- Tyler Burleigh, [Research, Plan, Implement, Review](https://tylerburleigh.com/blog/2026/02/22) (2026)
- avanwyk, [An Opinionated Agentic Engineering Workflow](https://avanwyk.com/an-opinionated-agentic-engineering-workflow/) (2026)
- Simon Willison, [Agentic Engineering Patterns](https://simonwillison.net/guides/agentic-engineering-patterns/)
- [agent-retro-omp](https://github.com/ankitg12/agent-retro-omp) — clock-aligned retro for OMP
- [chat-timestamps](https://github.com/ankitg12/chat-timestamps) — temporal instrumentation for OMP

---

*This post was written during a PAP session. The retro fired six times.*
