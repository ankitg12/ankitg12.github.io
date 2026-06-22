---
layout: post
title: "Auto-sync agent repos at session start"
date: 2026-06-22
categories: ai agents productivity
series: "AI coding agent productivity"
---

I run AI agents across two machines. Each agent has its own git repo. Logseq is git-backed too. Within a few weeks, the repos diverged: one machine pushed nine commits the other never pulled. The next push attempt hit an 11-commit split, a bad committed file (`C:\tmp\omp-screenshots\omp-screenshot.log` as a literal git path), and a rebase that couldn't run because Windows can't check out absolute-path filenames.

The fix took 25 minutes. The prevention takes one extension.

## The problem

Multi-device agent setups break in a specific way: you don't notice divergence until you try to push, by which point the merge is complex. Unlike a shared codebase, agent repos grow linearly — session notes, identity updates, knowledge files — and both devices produce valid content simultaneously. A standard merge loses nothing but is still friction.

The root cause is always the same: no pull before write.

## sync-repos.py

A single Python script handles all repos. It runs on both Windows and Mac without modification:

```python
REPOS = [
    (HOME / "Logseq",                                          "merge"),    # multi-device content
    (HOME / "repos/github.com/you/omp-sync-repos",             "ff-only"),  # this extension — self-updating
    (HOME / "agents" / "agent-a",                              "ff-only"),  # personal, single-author
    (HOME / "agents" / "agent-b",                              "ff-only"),
    # add your agent repos here
]
```

Two strategies:

| Strategy | When | Behaviour on divergence |
|---|---|---|
| `ff-only` | Personal single-owner repos | Warns loudly, doesn't auto-merge |
| `merge` | Logseq (daily journals) | Merges both sides; add/add conflicts resolved by keeping both |

A stamp file (`~/.omp/agent/sync-repos.stamp`) throttles to once per 30 minutes, so opening a new terminal tab doesn't trigger a pull:

```python
if not args.force and STAMP.exists():
    if time.time() - STAMP.stat().st_mtime < THROTTLE_SECS:
        return  # silent skip
```

`--force` bypasses the throttle. `--quiet` suppresses "already up to date" lines.

## OMP extension

OMP exposes a `session_start` event. A minimal extension fires the sync on every session regardless of terminal (WezTerm, Ghostty, whatever):

```ts
import type { ExtensionAPI } from "@oh-my-pi/pi-coding-agent";
import { execFile } from "child_process";
import { homedir } from "os";
import path from "path";

export default function syncReposExtension(pi: ExtensionAPI) {
  pi.setLabel("sync-repos");

  pi.on("session_start", async (_event, _ctx) => {
    const script = path.join(homedir(), "tools", "sync-repos.py");
    const python = process.platform === "win32" ? "python" : "python3";
    // Fire-and-forget — sync failure must never block a session
    execFile(python, [script, "--quiet"], { timeout: 30_000 }, () => {});
  });
}
```

Key decisions:

- **`execFile` not `pi.exec`** — `pi.exec` routes through brush (the OMP shell layer). `execFile` calls the OS Python directly, avoiding shell quoting issues on Windows.
- **Fire-and-forget** — the callback is empty. A failed pull is a warning, not a session blocker.
- **30s timeout** — prevents a hung SSH agent from stalling session start indefinitely.
- **Self-updating** — include the extension's own repo in `sync-repos.py`. Changes pushed from one machine arrive on the other at the next session start.

Register in `~/.omp/agent/config.yml`:

```yaml
extensions:
  - ~/repos/github.com/you/omp-sync-repos   # first — runs before other extensions
```

## Claude Code

Claude Code has the same hook point. Add to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python ~/tools/sync-repos.py --quiet || true"
          }
        ]
      }
    ]
  }
}
```

Empty `matcher` fires on all session starts (new and resumed). `|| true` ensures a sync failure never surfaces as a hook error.

## What this doesn't solve

- **Mid-session conflicts** — if you switch devices without closing the session, you'll still diverge within that session. The hook only fires at start.
- **Logseq add/add conflicts** — the merge strategy gets you past the conflict, but you still need to resolve which side's content goes first. A convention (remote content first, local retro notes after) handles this in practice.

## Source

- [`ankitg12/omp-sync-repos`](https://github.com/ankitg12/omp-sync-repos) — the OMP extension
