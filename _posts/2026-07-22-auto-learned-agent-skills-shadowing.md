---
layout: post
title: "When your agent's auto-learned skills start competing with each other"
date: 2026-07-22
categories: ai agents productivity tools
series: "AI coding agent productivity"
---

I asked my coding agent to audit its own configuration, and the single worst finding was self-inflicted: **108 auto-generated skills, only 2 of which I wrote by hand.** The rest had been minted silently by the agent's own "learn as you go" feature — and a good fraction of them were near-duplicates competing to answer the same question.

This is a failure mode specific to agents that can write their own skills. If yours does (OMP's `autolearn`, or any equivalent that persists `SKILL.md` files), you are probably accumulating the same debt without seeing it.

---

## What a skill is, and why duplication hurts

A "skill" here is a small markdown file with frontmatter — a name, a one-line `description:` that says *when to use it*, and a body with the actual procedure. The agent loads every skill's name + description into context on every turn, then picks the relevant one when a task matches.

That description line is the whole game. It's how the agent decides which skill fires. And it's always in context, whether or not you use the skill that session.

So duplication costs you twice:

1. **Context tax** — every description is always-loaded. My 108 skills were ~5.6k tokens of descriptions alone, paid on every single turn.
2. **Selection tax** — this is the subtle one. When five skills all say "use me for parsing that report," the agent has to disambiguate between near-identical options. More overlapping choices means worse picks. This is the same "too many tools degrades tool-use accuracy" effect documented for MCP tool sprawl — skills are just tools by another name.

The second tax is why this matters more than "some wasted tokens." The feature designed to make the agent *smarter over time* was quietly making it *worse at choosing*.

---

## How it happens

Auto-learn is greedy and has no memory of what it already knows. Each time the agent solves a fiddly problem, it may decide "this is worth codifying" and write a new skill. It does **not** check whether a skill covering that ground already exists. So over weeks you get clusters like:

```
report-parse
report-parse-fix
report-parsing
report-truncation-fix
report-triage
```

Five files. One actual topic. ~90% identical bodies, each written on a different day when the agent forgot it had already learned this.

The tell is the naming. **Cluster your skills by name prefix — any prefix with two or more entries is a duplication suspect.**

```python
import glob, os
from collections import defaultdict

skills = [os.path.basename(os.path.dirname(f))
          for f in glob.glob(os.path.expanduser("~/.omp/agent/managed-skills/*/SKILL.md"))]

clusters = defaultdict(list)
for name in skills:
    clusters["-".join(name.split("-")[:2])].append(name)

for prefix, members in sorted(clusters.items(), key=lambda x: -len(x[1])):
    if len(members) >= 2:
        print(f"[{len(members)}] {prefix}")
        for m in sorted(members):
            print("    ", m)
```

Prefix collision is a heuristic, not proof — but it turns "I have 100+ skills and no idea which overlap" into a ranked worklist in one pass.

---

## The consolidation protocol

Not every same-prefix cluster is a duplicate, and this is where a careless cleanup destroys knowledge. The rule that kept it lossless:

**A cluster is one of two things.**

**Near-duplicates** (≈80%+ shared body, same trigger). Collapse to one:

1. Pick the **superset** skill as the keeper — the one that already covers the most ground.
2. Read the others and extract any *unique* one-liner — a caveat, a gotcha, a distinct step — that the keeper is missing.
3. Fold those gems into the keeper.
4. **Only then** delete the redundant skills.

The ordering is the whole discipline: never delete until the unique content is preserved somewhere. Merging is lossy by default; you make it lossless by hand.

**Genuinely distinct siblings** that happen to share a prefix. Keep them all — but fix the descriptions:

| Skill | Bad description (shadows) | Fixed description |
|-------|---------------------------|-------------------|
| `x-breakout-triage` | "diagnose x port issues" | "capability triage: can the port reach target speed" |
| `x-readiness-gate` | "diagnose x port issues" | "config-to-runtime gate that drops the change" |
| `x-config-fix` | "diagnose x port issues" | "durable fix: edit config, regenerate, reboot" |

Same-prefix ≠ duplicate. Four skills about four *different* facets of one subsystem should stay four skills — but each description needs a unique trigger and, ideally, an explicit "NOT for X — see sibling" pointer so they stop competing.

---

## Parallelize the judgement

The assessment ("is this cluster a merge or a keep-distinct?") is real work, and it's embarrassingly parallel: each cluster is independent, and if you shard by name prefix, no two workers touch the same files.

I handed each prefix cluster to a separate subagent with one shared instruction set (the protocol above) and let them run concurrently. One pass took the library from 110 skills to 95 with zero content loss — and the subagents caught two bugs *inside the surviving skills* along the way (a reference to a script that no longer existed, and a stale command superseded months ago). Fresh eyes on skills you stopped reading are a bonus dividend.

A caution worth stating: run the *judgement* in parallel, but keep the *decomposition* — which clusters exist, how to shard them — as your own job. Sharding is the prerequisite every worker depends on; it can't itself be parallelized away.

---

## The part I can't fix yet

Nothing stops recurrence. Auto-learn keeps minting, so this is a periodic pass, not a one-time cleanup — the same way a memory store needs occasional compaction, not a single dedup.

The industry direction that would actually fix it is **retrieval-based loading**: instead of putting all N skill descriptions in context every turn, embed them and retrieve only the top-k relevant to the current task. That caps both the context tax and the selection tax regardless of library size. The [Toolshed paper](https://arxiv.org/abs/2410.14594) works through exactly this for tool-equipped agents. Until agent frameworks ship dedup-on-write or retrieval-on-demand, a scheduled consolidation pass is the pragmatic stopgap.

The broader lesson is one worth internalizing about any self-improving system: **a feature that accumulates without ever consolidating will eventually degrade the thing it was meant to improve.** Measure it, or it compounds against you.

---

## Source

- [Toolshed: Scale Tool-Equipped Agents with Advanced RAG-Tool Fusion](https://arxiv.org/abs/2410.14594) — retrieval-based tool/skill loading
- [Anthropic — Writing effective tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents) — why tool/skill descriptions are the selection surface
- [Anthropic — Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — the always-loaded-context cost model
