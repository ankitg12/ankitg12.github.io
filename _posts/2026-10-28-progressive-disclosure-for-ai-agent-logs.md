---
layout: post
title: "Progressive Disclosure for AI Agent Logs: Preventing Context Window Bloat at Git Pre-Commit"
date: 2026-10-28
categories: ai agents productivity tools
---

An AI coding agent tasked with investigating a flaky system test ran twenty terminal commands, pasted six `dmesg` excerpts, and logged every raw step into `notes/2026-10-28.md`—turning what should have been a five-minute handoff into a 1,800-line markdown monolith that poisoned every subsequent agent session in the repository.

When an agent starts a session, it reads repository documentation and recent session logs to rebuild state. If logs grow without bounds, the agent's context window fills with stale terminal outputs, token costs multiply, attention across tool calls degrades, and state handoffs hallucinate details from weeks ago.

Deleting logs destroys provenance; keeping everything destroys context economy. The solution is filesystem-level information hiding: a four-tier logging hierarchy (L0–L3) with automatic compaction enforced at git pre-commit.

---

## The Four Log Tiers (L0–L3)

Following David L. Parnas' principle of information hiding (CACM, 1972), each tier exposes only the resolution of detail needed for its consumer:

| Tier | File / Directory | Content | Budget | Rotation Policy |
|---|---|---|---|---|
| **L0 Now** | `RUNLOG.md` (`## Now`) | Live pointers, current task, active blockers. | $\le 15$ lines | **Overwritten** each turn; never appended. |
| **L1 Index** | `RUNLOG.md` (`## Index`) | Milestone index: `- YYYY-MM-DD <milestone>`. | $\le 60$ lines, $\le 120$ chars/line | Oldest lines automatically rotated to `runlog/YYYY-MM.md`. |
| **L2 Day Note** | `notes/YYYY-MM-DD.md` | Working narrative. First section must be `## Summary` ($\le 10$ lines). | Whole file $\le 120$ lines | Excess detail sections rotated verbatim to `archive/notes/YYYY-MM-DD.md`. |
| **L2 Archive** | `archive/notes/YYYY-MM-DD.md` | Full raw detail moved out of day notes. | Unlimited | Not loaded at session start; grep on demand. |
| **L3 Evidence** | `artifacts/`, external trackers | Raw pcaps, dmesg dumps, large test outputs. | Unlimited | Referenced by URL or relative path from L2. |

When a new agent session starts, it loads:
1. `RUNLOG.md` `## Now` (first 15 lines) for the immediate operational boundary.
2. The `## Summary` (first 10 lines) of the latest day note for recent progress.
3. If investigating an older topic, it greps `runlog/` or `archive/notes/` rather than reading raw files into context.

---

## Why Linters and Logrotate Don't Solve This

Traditional tools address pieces of this problem, but none solve progressive disclosure for Markdown:

1. **Markdown linters** (`markdownlint`, `remark-lint`):
   They can flag lines longer than 120 characters or sections exceeding line budgets. But they are passive: they fail CI and force human friction, rather than actively rotating and preserving history.
2. **System log rotators** (`logrotate`, `newsyslog`):
   They treat files as opaque byte streams, rotating `app.log` to `app.log.1`. They cannot parse markdown ASTs, preserve an executive `## Summary`, or extract and link sections to an archive table of contents.
3. **Agent memory prompts**:
   Prompting an LLM to "keep notes under 120 lines" fails predictably under complex debugging pressure. Invariants must be enforced deterministically by the harness, not by the model under test.

---

## How Automatic Compaction Works

The tool (`log-tiers`) runs as a git pre-commit hook with `--compact --stage`. When a commit is triggered:

```text
git commit
   │
   ├── 1. Check notes/<date>.md line count
   │      └── If > 120 lines:
   │            - Keep preamble and ## Summary (< 10 lines)
   │            - Move all subsequent ## detail sections verbatim to archive/notes/<date>.md
   │            - Replace moved sections with an index:
   │                ## Detail
   │                Moved verbatim to [archive/notes/...]; grep there by heading.
   │                - Heading 1
   │                - Heading 2
   │
   ├── 2. Check RUNLOG.md ## Index count
   │      └── If > 60 lines:
   │            - Move oldest milestone lines to runlog/YYYY-MM.md (newest first)
   │
   ├── 3. Enforce Semantic Invariants:
   │      - Freshness: Adding a new detail heading to a staged note requires updating ## Summary
   │      - Strictness: RUNLOG.md must contain only ## Now and ## Index
   │      - Chronology: Index entries must be strictly sorted newest-first
   │
   └── 4. Automatically `git add` modified archives and runlogs
```

Nothing is deleted. History is preserved verbatim in `archive/notes/`, while active context files remain under a strict token ceiling.

---

## Configuration (`.log-tiers.toml`)

Repositories configure limits and custom paths via standard TOML (zero external dependencies using Python 3.11's built-in `tomllib`):

```toml
# .log-tiers.toml

title = "My Project Run Log"

# Tier limits
now_max = 15            # Maximum lines in RUNLOG.md ## Now
index_max = 60          # Maximum lines in RUNLOG.md ## Index
line_max = 120          # Maximum characters per ## Index line
summary_max = 10        # Maximum lines in notes/<date>.md ## Summary
notes_max = 120         # Maximum lines in a day note before rotation
heading_max = 110       # Max characters for archived heading summary

# File paths
runlog_path = "RUNLOG.md"
notes_dir = "notes"
runlog_dir = "runlog"
archive_notes_dir = "archive/notes"
detail_heading = "Detail"
```

---

## Installation & Setup

Install globally via `uv`:

```bash
uv tool install git+https://github.com/ankitg12/ankitg-tools.git --subpath log-tiers
```

Or clone locally:

```bash
git clone https://github.com/ankitg12/ankitg-tools.git ~/tools/ankitg-tools
uv tool install --editable ~/tools/ankitg-tools/log-tiers
```

Wire into any git repository's pre-commit hook:

```bash
mkdir -p .githooks
cat << 'EOF' > .githooks/pre-commit
#!/bin/sh
root="$(git rev-parse --show-toplevel)"

if command -v log-tiers >/dev/null 2>&1; then
    exec log-tiers --compact --stage "$root"
else
    echo "log-tiers: command not found. Install via 'uv tool install log-tiers'." >&2
    exit 1
fi
EOF
chmod +x .githooks/pre-commit
git config core.hooksPath .githooks
```

---

## Source & References

- Source Repository: [ankitg12/ankitg-tools/log-tiers](https://github.com/ankitg12/ankitg-tools/tree/master/log-tiers)
- Architectural Lineage: David L. Parnas, *On the Criteria To Be Used in Decomposing Systems into Modules* (CACM, December 1972, Vol. 15, No. 12).
