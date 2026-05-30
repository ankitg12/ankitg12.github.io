---
layout: post
title: "Mining AI agent session logs for skill candidates"
date: 2026-05-30
categories: ai agents productivity tools
series: "AI coding agent productivity"
---

[crune](https://dev.to/chigichan24/mining-hidden-skills-from-claude-code-session-logs-with-semantic-knowledge-graphs-2em8) is a great idea: analyze your Claude Code session JSONL logs, build a semantic knowledge graph, and surface reusable skill candidates — workflows you do repeatedly but haven't codified. The problem is the data source. If you use more than one agent, crune doesn't help.

I run four agents — OMP (Pi fork), Claude Code, Codex, and occasionally Gemini. crune found 216 of my OMP sessions and tokenized exactly 0 of them. OMP wraps messages as `{"type": "message", "message": {"role": "..."}}` while crune expects `{"type": "user"}` at the top level. Directory structure is also different. A shim is ~50 lines, but that still leaves codex and gemini out.

The right data source is [agentsview](https://agentsview.io) — a session analytics tool that already normalizes across agents. It reads whatever JSONL each agent produces and stores everything in a single SQLite database. 910 sessions across five agents in mine.

## What agentsview already knows

The `sessions` table has pre-computed signals that crune would need ML to discover:

| Column | What it measures |
|---|---|
| `tool_retry_count` | Tool calls that needed retrying |
| `consecutive_failure_max` | Longest failure streak |
| `tool_failure_signal_count` | Total tool failures |
| `edit_churn_count` | Edit-then-revert cycles |
| `health_score` / `health_grade` | Overall session quality |
| `outcome` | `success` / `failure` / `unknown` |

The `tool_calls` table has 50,000+ rows with `tool_name`, `category`, `input_json`, and `result_content`. Tool co-occurrence across sessions is a better workflow signal than TF-IDF on message content — it's structural, not lexical.

## avmine.py

Three queries do the work:

**Friction hotspots** — projects with highest avg retry rate. These are where skills would reduce overhead most:

```sql
SELECT s.project, s.agent, COUNT(*) AS n_sessions,
       ROUND(AVG(s.tool_retry_count), 2) AS avg_retry,
       MAX(s.consecutive_failure_max) AS max_fail
FROM sessions s
WHERE s.project != ''
GROUP BY s.project
HAVING n_sessions >= 3
ORDER BY avg_retry DESC
```

**Workflow fingerprints** — top tools per project, normalized to lowercase to merge `bash`/`Bash`, `read`/`Read` etc. across agents:

```sql
SELECT s.project, lower(tc.tool_name) AS tool,
       COUNT(DISTINCT s.id) AS sessions
FROM sessions s JOIN tool_calls tc ON tc.session_id = s.id
GROUP BY s.project, lower(tc.tool_name)
HAVING sessions >= 3
ORDER BY s.project, sessions DESC
```

**Tool pairs** — co-occurrence within the same session. Each pair is a workflow unit:

```sql
SELECT lower(t1.tool_name) AS tool_a, lower(t2.tool_name) AS tool_b,
       COUNT(DISTINCT t1.session_id) AS sessions
FROM tool_calls t1
JOIN tool_calls t2 ON t1.session_id = t2.session_id
    AND lower(t1.tool_name) < lower(t2.tool_name)
JOIN sessions s ON s.id = t1.session_id
GROUP BY lower(t1.tool_name), lower(t2.tool_name)
HAVING sessions >= 4
ORDER BY sessions DESC
```

Skill candidates are ranked by `sessions × (1 + avg_retry)` — high recurrence plus friction means the most opportunity.

## LLM synthesis

The raw output names patterns by tool combination: `read+bash+web_search`. That's accurate but not useful. A single Anthropic API call (claude-3-5-haiku-20241022 for speed, Opus for quality) turns it into something actionable:

Without synthesis:
```
tmp [pi, 97 sessions]  score=97.0
tools: read + bash + web_search
```

With Haiku:
```
### quick-filesystem-search
Rapidly locate and inspect files using read + bash + web lookups.
Trigger: "find this file", "search the codebase", "where is X defined"
Evidence: 97 sessions, 0.0 retry — stable, well-understood, repeatable.
```

With Opus:
```
### web-augmented-scripting
Research topics via web search, read relevant files, and execute bash scripts
to accomplish ad-hoc tasks in temporary/scratch projects.
Trigger: "look up and script", "search and run", "find how to do X and implement it"
Evidence: 97 sessions, highest volume, zero retry — frictionless, ideal for codification.
```

The `--full` flag generates complete `SKILL.md` files ready to drop into `~/.claude/skills/`.

## Usage

```bash
pip install agentsview   # if not already installed

# Full session history
python avmine.py --top 10 --synthesize

# Today only
python avmine.py --since 2026-05-30 --min-sessions 1 --top 5

# Complete SKILL.md files
python avmine.py --top 3 --full --model claude-opus-4-5

# Fast/cheap with Haiku
python avmine.py --top 5 --synthesize --model claude-3-5-haiku-20241022

# Filter by agent
python avmine.py --agent pi --synthesize
```

All flags:

| Flag | Default | Purpose |
|---|---|---|
| `--db` | `~/.agentsview/sessions.db` | Path to agentsview DB |
| `--agent` | all | Filter by agent: `claude`, `pi`, `codex`, `gemini`, `opencode` |
| `--since` | all time | Start date `YYYY-MM-DD` |
| `--min-sessions` | 4 | Minimum sessions for a pattern to surface |
| `--top` | 10 | Top N candidates |
| `--synthesize` | off | Generate skill stubs via Anthropic API |
| `--full` | off | Generate complete SKILL.md files (implies synthesis) |
| `--model` | `claude-opus-4-5` | Anthropic API model |

API key: reads from `ANTHROPIC_API_KEY` env var or `~/.omp/agent/models.yml` (`x-api-key`).

## What it surfaces

On 910 sessions across 5 agents:

```
Friction Hotspots
  computer_agent   [pi]     9 sessions   avg_retry=0.44
  Projects         [claude] 26 sessions  avg_retry=0.15
  angaur           [pi]    125 sessions  avg_retry=0.07

Top Skill Candidates
  1. tmp            [pi]     score=97   tools: read+bash+web_search
  2. gpu_operator_qa [claude] score=56  tools: read+edit+bash
  3. angaur         [claude] score=44.9 tools: bash+shell_command+read
```

`computer_agent` has the highest friction per session — 44% of sessions needed tool retries. That's where a skill would help most. `gpu_operator_qa` at 56 sessions with zero retries is a mature, repeatable workflow begging to be codified.

## Source

`avmine.py` is available as a [GitHub Gist](https://gist.github.com/ankitg12/a058f959a7b67a5a42d2f845445e299d) and in [ankitg12/ankitg-tools](https://github.com/ankitg12/ankitg-tools). No dependencies beyond stdlib — just `sqlite3`, `urllib.request`, `json`, `argparse`. agentsview must be installed and synced (`agentsview sync`) before running.
