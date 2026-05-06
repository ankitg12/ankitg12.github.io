---
layout: post
title: "Migrating 40 repos to ghq on Windows: what went wrong and what fixed it"
date: 2026-05-06
categories: git tools windows productivity
---

[ghq](https://github.com/x-motemen/ghq) organises local git clones into a `host/owner/repo` tree under a single root. The idea is borrowed from Go's module layout: clone `github.com/org/project` and it lands at `~/repos/github.com/org/project`, always, across every machine. `ghq list` gives you everything; `ghq list -p | fzf` lets you jump to any repo in two keystrokes.

The starting state on my machine: `ghq list` returned two entries. The actual count across `~/repos`, `~/workspace`, and `~/tools` was 40.

---

## Why the repos were scattered

No single cause. Over time repos accumulate wherever you clone them. `git clone` at the root of `~/workspace` for work repos. Ad-hoc clones into `~/repos` for tools you're reading. Standalone scripts in `~/tools` that happen to be git repos. None of it conforms to any structure.

The result: you forget where things are, navigation aliases rot, and nothing shows up in `ghq list`.

---

## The wrong first move

My first instinct was a PowerShell script: scan both directories, parse remote URLs, compute canonical ghq paths, move directories, handle duplicates. I got 200 lines in before hitting the first problem.

PowerShell inline via bash mangles `$` variables before the shell sees them. Using `-Command "..."` with embedded scriptblocks hits quoting limits fast. Writing the script to a `.ps1` file and using `-File` is the only reliable path — and that rule was already in my notes.

The encoding bug came next. An em-dash in a string literal (`—`) gets corrupted when the file is written by a tool with a different default encoding than PowerShell expects. The parser produces `unexpected token 'manual'` with no obvious connection to the cause.

Then `$ErrorActionPreference = 'Stop'` converted git's stderr warnings into thrown exceptions, breaking the dirty-tree check on large repos.

Three rounds of fixes. Nothing shipped.

---

## The right approach: `ghq migrate`

Before writing a single line, I should have run:

```sh
ghq --help
ghq migrate --help
```

`ghq migrate` has existed since v1.10 (February 2026):

```
NAME:
    migrate - Migrate existing repository to ghq-managed directory

USAGE:
    ghq migrate [-y] [--dry-run] <repository-directory>
```

It reads the remote URL, derives the canonical path, creates parent directories, and moves the repo atomically. One command per repo. The entire migration script reduces to a loop.

---

## The bulk migration script

With `ghq migrate` as the primitive, the script becomes coordination logic rather than mechanics:

```python
# Collect all top-level git repos from multiple source dirs
SOURCES = [
    GHQ_ROOT,                   # flat clones that predate ghq
    GHQ_ROOT / "work-org",      # leftover 2-deep partial structure
    Path.home() / "workspace",  # parallel working copies
    Path.home() / "tools",      # standalone repos alongside scripts
]
```

For each unique repo (keyed by `host/owner/repo` parsed from the remote URL):

1. If multiple copies exist, the workspace copy wins; tiebreak on newest commit.
2. Only check dirty status on *losers* (repos to be deleted), not on winners being moved. Moving a repo with uncommitted changes is safe — `.git` travels intact. Dirty-checking winners adds O(n) `git status` calls on large repos like the Linux kernel or PowerShell source.
3. Run `ghq migrate -y <winner>`, then delete clean losers.

The script is in the references at the end if you want the full thing.

---

## Windows-specific problems

### git object files are read-only

Git marks object files read-only on Windows as a content-addressable store integrity measure. `shutil.rmtree` raises `PermissionError` when it hits one. Fix:

```python
def rmtree_force(path: Path) -> None:
    def _on_err(func, p, _exc):
        Path(p).chmod(stat.S_IWRITE)
        func(p)
    shutil.rmtree(path, onerror=_on_err)
```

### Handle release delay after process kill

`ghq migrate` uses `os.Rename` → `MoveFileExW`. This requires an exclusive lock on the directory entry. If a process holds any handle inside the tree, you get `Access is denied`.

After `Stop-Process` or `taskkill`, Windows releases handles asynchronously. Retrying `MoveFileExW` immediately after the kill still sees the handle. Wait ~15 seconds, retry, and it succeeds.

If the window is tight, robocopy + rmtree bypasses the issue entirely:

```powershell
# robocopy exit 1 = "files copied successfully" — not an error
robocopy C:\src C:\dst /E /NFL /NDL /NJH /NJS
python -c "
import shutil, stat
from pathlib import Path
def f(fn, p, _): Path(p).chmod(stat.S_IWRITE); fn(p)
shutil.rmtree('C:/src', onerror=f)
"
```

Robocopy reads files one-by-one with shared access. `rmtree` runs after enough time has passed for handles to drain.

### Do not migrate system config directories

Running `ghq migrate ~/Documents/PowerShell` moves the directory into the ghq root. PowerShell loads `$PROFILE` from a fixed path; moving the directory breaks the load chain silently.

The rule: `ghq migrate` is for source code repos. Config directories need the inverse pattern — keep the file in the repo, symlink from the expected system path:

```cmd
mkdir %USERPROFILE%\Documents\PowerShell
mklink %USERPROFILE%\Documents\PowerShell\Microsoft.PowerShell_profile.ps1 ^
       %USERPROFILE%\repos\github.com\user\dotfiles\Microsoft.PowerShell_profile.ps1
```

---

## Preserving unique changes in duplicate repos

The same repo cloned in two places diverges in the working tree. Before deleting the stale copy, check what's unique:

```sh
git -C /stale/copy status --short      # M Dockerfile
git -C /canonical/copy status --short  # nothing
```

Apply only the delta:

```sh
git -C /stale/copy diff Dockerfile | git -C /canonical/copy apply -
```

No temp files, no cherry-picking, no manual merging. Pipe the diff directly.

---

## Repos in unexpected places

`~/tools/` had four git repos alongside standalone scripts — a break-timer sync tool, a PowerShell helper module, an agent, a chat timestamp exporter. None showed in `ghq list` because `~/tools/` was not a scan source.

Quick audit:

```bash
for d in ~/tools/*/; do
  remote=$(git -C "$d" remote get-url origin 2>NUL)
  [ -n "$remote" ] && echo "$d => $remote"
done
```

Add discovered sources to `SOURCES` in the migration script, re-run, done.

---

## Beyond migration: gwq and parallel agents

Once repos are in the right place, the next question is worktrees. [gwq](https://github.com/d-kuro/gwq) manages worktrees the same way ghq manages repos — canonical naming, fuzzy-finder navigation, same root directory.

Configure gwq to use the same root:

```toml
# ~/.config/gwq/config.toml
[naming]
template = '{{.Host}}/{{.Owner}}/{{.Repository}}={{.Branch}}'

[worktree]
basedir = '~/repos'
```

With this layout, a feature branch for a repo lands at:
```
~/repos/github.com/org/project=feat-x/
```
alongside the main clone at `~/repos/github.com/org/project`. One `ghq list -p | fzf` covers repos and active branches simultaneously.

This matters for AI coding agents. Each agent session needs an isolated working directory — the same repo under active edit by two sessions produces conflicts. Worktrees are the right primitive: separate HEAD, index, and untracked files, but a single shared object store. One `git fetch` updates everything.

```sh
gwq add -b agent-task-a   # agent A
gwq add -b agent-task-b   # agent B
# two terminal sessions, each cd'd into its own worktree
```

Anthropic's own best practices for Claude Code recommend this pattern.

One caveat on Windows: `worktree.useRelativePaths = true` (Git 2.46+) is worth setting globally to prevent stale absolute paths when repos move. But oh-my-posh 29.x has a bug where relative worktree paths break the git segment inside linked worktrees. If the git prompt goes blank when you `cd` into a worktree, that's the cause:

```sh
git config --global --unset worktree.useRelativePaths
```

---

## Summary

| Problem | Solution |
|---|---|
| `ghq list` shows 2 of 40 repos | `ghq migrate -y <path>` — bulk via `ghq-migrate.py` |
| PS script encoding/ErrorActionPreference | Use Python; pass PS via `-File`, never `-Command` |
| `shutil.rmtree` fails on git objects | `onerror` handler that `chmod`s before delete |
| `MoveFileExW` fails after process kill | Wait ~15s; fallback: robocopy + rmtree |
| Repos in `~/tools/` not scanned | Extend `SOURCES` list; re-run |
| Duplicate repos with unique changes | `git diff | git apply -` between copies |
| Config dir migrated → broken profile | Symlink FROM expected path TO repo, not the reverse |
| Multiple agents colliding on one branch | `gwq add -b <branch>` per agent; shared object store |

---

## References

- [ghq](https://github.com/x-motemen/ghq) — repo manager; `ghq migrate` since v1.10
- [gwq](https://github.com/d-kuro/gwq) — worktree manager, same conventions as ghq
- [PSFzf](https://github.com/kelleyma49/PSFzf) — fzf keybindings for PowerShell (`Ctrl+T`, `Ctrl+R`)
- [ghq + gwq + fzf for AI agents](https://dev.to/shunk031/a-coding-agent-friendly-environment-is-friendly-to-humans-too-ghq-gwq-fzf-2km0) — the article that surfaced gwq
- [git worktree and parallel AI agents](https://recca0120.github.io/en/2026/04/14/git-worktree-parallel-work/) — worktree layouts for multi-agent workflows
- [chezmoi](https://chezmoi.io) — dotfiles manager when the symlink approach scales out
