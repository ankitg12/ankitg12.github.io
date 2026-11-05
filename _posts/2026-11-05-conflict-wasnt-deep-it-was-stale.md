---
layout: post
title: "The Conflict Wasn't Deep. It Was Stale."
date: 2026-11-05
categories: git productivity debugging
series: "AI coding agent productivity"
---

A three-paragraph notes conflict ate an hour this morning. Not because it was hard — because the failure kept disguising itself as a fresh one, and I kept merging against a target that had already moved. The fix was `git reset --hard origin/master`. The lesson is knowing when to stop merging and restart.

## The symptom that lied

`git pull` failed with "Pulling is not possible because you have unmerged files." Fair enough — except it failed the same way yesterday, and the day presents it as a new event. What had actually happened: yesterday's pull started a `rebase`, the autostash pop conflicted on one file, and git suspended the rebase mid-flight. Every later pull just re-reported the suspension.

The tell is a 30-second check most people skip:

```bash
git status --short          # UU = unmerged, the real blocker
ls .git/rebase-merge        # exists = a rebase is suspended, NOT a fresh failure
cat .git/rebase-merge/done  # non-empty = commits already applied; only the stash pop is stuck
```

If `done` is non-empty, you have one conflict, not many. Everything downstream of that realization gets cheaper.

## The moving target

While I merged, `origin/master` advanced four commits — blog index rows landing in the exact region I was hand-editing. Fix one diff, fetch, find another. This is the signature of merging against a moving target: the diffs multiply, but the content at stake stays constant. Mine was three paragraphs across three files. The dozens of diffs I stared at were the same lines viewed from different angles — staged vs unstaged vs stash vs origin vs an old baseline.

The rule, stated once: if upstream moved *into* your conflicted files, stop resolving forward. Save your blobs, abort, reset, re-apply. One reset beats three merges.

```bash
git show <commit>:<file> > /tmp/save-<file>   # your work, safe
git rebase --abort
git reset --hard origin/master
git pull   # "Already up to date."
```

Recovery commits survive in the reflog until gc, so the reset is reversible. The blobs make the re-apply mechanical.

## The self-inflicted fog

Honestly reported: half the hour was mine. I merged a bullet, reordered index rows to match what I *thought* was upstream order, renamed a file mid-rebase — each step changed the worktree and made the next diff harder to read. At one point I deleted pushed rows thinking they were local, then restored them one by one. Editing while merging is context arson; the tree should be frozen except for the conflict hunks.

Two smaller gotchas from the same session, for the record: a `core.hooksPath=/dev/null` plus a stray `.pre-commit-config.yaml` made every commit fail with `InvalidConfigError` (fixed with `--no-verify`, then fix the config); and verifying Chromebook/Linux paths from macOS proved nothing, because `~/.local/share/applications/` doesn't exist on macOS. Check claims on the machine they describe.

## The lineage

None of this is new. `pull --rebase` with a dirty tree — stash, rebase, pop — has suspended this way since the option existed, which is why Magit (Jonas Bernoulli, 2008) never pulls implicitly and why GitHub-flow (Scott Chacon, *Pro Git*, 2009) keeps work on branches so `main` can move underneath you. `git worktree` (Git 2.5, 2015) exists so archaeology happens in a second directory instead of inside the live tree. I did worktree-shaped work inside the live tree, and paid for it.

The post-worthy part isn't the git. It's the meta-pattern: suspended state masquerading as fresh failure, forward effort against a moving target, and the discipline of resetting early. An hour, three paragraphs, one reset. Next time: one pass.
