---
layout: post
title: "269 Lines of Python for a Tool Already Installed: Extending lychee Instead of Reinventing It"
date: 2026-11-28
categories: ai agents productivity
series: "AI coding agent productivity"
---

My coding agent wrote a 269-line Python link checker to audit broken file paths in my notes repository — while [lychee](https://lychee.cli.rs/) was already installed on the machine and listed in the repository's own tools manifest.

The fix took three human interruptions in 15 minutes and ended with a 54-line `lychee` preprocessor plugin, a 16-line `lychee.toml`, and a global Git pre-commit hook. The interesting part is not the script. It is how an agent with an explicit "Don't Reinvent the Wheel" rule in its system prompt still bypassed an installed upstream tool — and why a thin plugin beat both extremes.

## What broke first: pointer rot and ad-hoc glue

The morning started with a routine daily briefing. The agent ran several ad-hoc shell loops and inline Python snippets to scan email, calendar, and notes. When I asked why it was writing throwaway scripts instead of using existing tools, two root causes surfaced:

1. **A dead pointer in long-term memory.** A memory entry pointed to `~/Notes/m365.md`, which had been moved into `~/Notes/tools-manifest/microsoft-365.md`.
2. **Split instructions.** One skill pointed to one CLI, memory pointed to another, and neither documented the newer filtering flags.

Fixing those pointers raised the obvious follow-up: *what other pointers are broken across the notes and agent repositories?*

## How the agent missed an installed upstream tool

I keep a "Don't Reinvent the Wheel" rule in my agent instructions: check installed tools and the local manifest before writing more than 30 lines of new code. Yet when asked to audit broken pointers, the agent still wrote `check_pointers.py` (269 lines of Python + unit tests + a symlink).

Two small failures combined:

1. **A dropped check.** The agent bundled `which lychee ...` with `head` in a single shell command. A tool policy rejected the command because of `head`. When the agent split the call, it read a local script and never re-ran `which lychee`.
2. **A skipped manifest grep.** `lychee` (`0.24.2`, managed via [mise](https://mise.jdx.dev/)) was listed under `editorial-checks.md` in the tools manifest, with the top-level index explicitly mentioning `link checks`. The agent had read the top-level index minutes earlier and moved straight to writing code.

The agent did not ignore the rule out of malice; it rationalized past it. Once a model starts picturing a custom parser, the shortest path to a green test feels faster than searching for an existing binary.

## Why bare `lychee --offline` was not enough either

When I interrupted — *"We do have a tool to check links, right?"* — the agent ran `lychee --offline --no-progress` on both repositories:

| Repository | Links checked | Errors reported by bare `lychee --offline` | Real broken `~/...` pointers |
|---|---:|---:|---:|
| Notes (active routers + manifest) | 416 | 0 | 5 |
| Agent workspace (active docs + skills) | 55 | 0 | 18 |

`lychee` was fast (19 ms and 4 ms) and completely blind to the actual failure mode. By default, `lychee` parses Markdown links (`[text](target)`) and HTML `<a href>` tags. Knowledge repositories and agent instruction files rarely write `[manifest](~/Notes/m365.md)` — they write backticked local paths like `` `~/Notes/m365.md` `` or `` `~/tools/some-script.py` `` inside prose and code fences.

That is where a second trap sits: it is easy to conclude *"upstream doesn't support our format, so we need our own 269-line scanner."*

## The second human pushback: check upstream extension points

I pushed back once more: *can we make this work through `lychee` via config or a plugin instead of maintaining our own checker?*

Checking `lychee --help` and the upstream [`lychee.example.toml`](https://github.com/lycheeverse/lychee/blob/master/lychee.example.toml) turned up two native features that solved the entire problem:

1. **`include_verbatim = true`** — tells `lychee` to extract links inside inline backticks and fenced code blocks.
2. **`preprocess = { command = "..." }`** — pipes each input file through an external command before link extraction.

`lychee --offline` already validates `file:///...` URIs natively, with exact `(line:col)` locations, `.gitignore` handling, `exclude_path`, custom scheme exclusions, and Rust speed. The only missing piece was turning concrete `~/path/to/file` tokens into `file://$HOME/path/to/file` while preserving line numbers 1-to-1 and skipping glob/template patterns (`YYYY_MM_DD`, `*.md`, `/<placeholder>`).

## The 54-line preprocessor + `lychee.toml`

Instead of 269 lines of custom file walking, path resolution, and formatting, the plugin (`~/tools/lychee-home-paths`) does one job:

```python
#!/usr/bin/env python3
"""Lychee --preprocess plugin: expand concrete ~/... paths to file://$HOME/... URLs."""

from __future__ import annotations

import os
import re
import sys
from pathlib import Path

HOME_URI = Path.home().as_uri().rstrip("/")
HOME_PATH_RE = re.compile(
    r"(?<![A-Za-z0-9_./:-])~/([A-Za-z0-9_.\-/+@]+)(?::\d+(?:-\d+)?)?"
)
TEMPLATE_TOKENS = (
    "YYYY", "MM", "DD", "<", ">", "*", "{", "}", "$", "...",
    "project-key", "/name", "/foo", "/bar", "example", "my-", "your-",
)


def rewrite_line(line: str) -> str:
    def _sub(m: re.Match[str]) -> str:
        if m.end() < len(line) and line[m.end()] in "*?[{<$":
            return m.group(0)
        if m.end() + 1 < len(line) and line[m.end():m.end() + 2] in ("/<", "/{", "/$", "/*"):
            return m.group(0)
        rel = m.group(1).rstrip(".,;:!?`'\"")
        rel = re.sub(r":\d+(?:-\d+)?$", "", rel)
        if not rel or any(tok in rel for tok in TEMPLATE_TOKENS) or re.search(r"/[A-Z]$", rel):
            return m.group(0)
        suffix = m.group(1)[len(rel):]
        return f"{HOME_URI}/{rel}{suffix}"

    return HOME_PATH_RE.sub(_sub, line)
```

Each repository then drops a `lychee.toml` at its root:

```toml
offline = true
include_verbatim = true
no_progress = true
preprocess = { command = "/home/user/tools/lychee-home-paths" }

extensions = ["md"]
exclude_path = [
    "archive/",
    "\\.git/",
]

exclude = [
    "^(omp|skill|memory|rule|local|mailto|tel):",
]
```

Running bare `lychee .` inside either repository now checks Markdown links, HTML links, and backticked `~/...` local paths in one pass — 158 links in 344 ms — and pinpoints every broken pointer by exact line and column. We deleted `check_pointers.py` and its symlink.

## Locking it in with a global Git pre-commit hook

Auditing once does not stop pointers from rotting next month. Rather than wiring hooks repo by repo, a shared script (`~/tools/git-hook-lychee`) checks only staged `.md` files whenever a repository has a root `lychee.toml`:

```bash
#!/usr/bin/env bash
set -euo pipefail

root=$(git rev-parse --show-toplevel 2>/dev/null) || exit 0
[ -f "$root/lychee.toml" ] || exit 0

mapfile -t staged < <(git diff --cached --name-only --diff-filter=ACM -- '*.md')
[ "${#staged[@]}" -eq 0 ] && exit 0

export PATH="$HOME/.local/share/mise/shims:$HOME/.local/bin:$PATH"
command -v lychee >/dev/null 2>&1 || exit 0

cd "$root"
exec lychee --config "$root/lychee.toml" -- "${staged[@]}"
```

Setting [Git's global `core.hooksPath`](https://git-scm.com/docs/git-config#Documentation/git-config.txt-corehooksPath) to `~/.config/git/hooks` with a wrapper that first chains any repo-local `.git/hooks/pre-commit` activates the check automatically for every repository that opts in with a `lychee.toml` (and repositories with their own local `core.hooksPath` simply call `~/tools/git-hook-lychee` from their existing hook).

| Approach | Code we maintain | Checks `[text](url)` + HTML | Checks backticked `~/...` | Pre-commit latency |
|---|---:|---|---|---|
| Bare `lychee --offline` | 0 lines | Yes | No | ~20 ms |
| Custom `check_pointers.py` | 269 lines | Partial | Yes | ~150 ms |
| `lychee` + `lychee-home-paths` plugin | 54 lines | Yes | Yes | ~100–350 ms |

## Why human intervention mattered

Three short questions changed the outcome completely:

1. *"Why write on-the-fly scripts instead of reusable tooling?"* — exposed broken pointers and missing CLI filters.
2. *"Don't we already have a link checker installed?"* — stopped a 269-line reinvention from becoming permanent technical debt.
3. *"Can we extend `lychee` via config or a plugin instead?"* — turned a custom tool into a 54-line upstream adapter and a global pre-commit guard.

Written rules like "Don't Reinvent the Wheel" lower the baseline, but agents still bias toward writing fresh code when an upstream tool does not match the exact input format out of the box. The cheapest engineering leverage is still asking: *what hook does the upstream tool already expose?*

## Source

- [lychee — Fast, async, resource-friendly link checker](https://lychee.cli.rs/)
- [lychee GitHub repository & `lychee.example.toml`](https://github.com/lycheeverse/lychee/blob/master/lychee.example.toml)
- [mise — polyglot tool version manager](https://mise.jdx.dev/)
- [Git `core.hooksPath` documentation](https://git-scm.com/docs/git-config#Documentation/git-config.txt-corehooksPath)
