---
layout: post
title: "Auto-sync agent repos at session start"
date: 2026-06-22
categories: ai agents productivity
series: "AI coding agent productivity"
---

I run AI agents across two machines. Each agent has its own git repo. Within a few weeks, repos diverged: one machine pushed nine commits the other never pulled. The next push attempt hit an 11-commit split, a bad committed file (`C:\tmp\omp-screenshots\omp-screenshot.log` as a literal git path), and a rebase that couldn't run because Windows can't check out absolute-path filenames.

The fix took 25 minutes. The prevention takes one extension.

## The problem

Multi-device agent setups break in a specific way: you don't notice divergence until you try to push. The root cause is always the same: no pull before write.

## The extension

An OMP `session_start` hook that pulls the current session's repo — nothing more:

```ts
// index.ts
import type { ExtensionAPI, SessionStartEvent } from "@oh-my-pi/pi-coding-agent";
import { execFile } from "child_process";
import path from "path";

const SCRIPT = path.join(__dirname, "sync-repos.py");
const PYTHON = process.platform === "win32" ? "python" : "python3";

export default function syncReposExtension(pi: ExtensionAPI) {
  pi.setLabel("sync-repos");

  pi.on("session_start", async (event: SessionStartEvent, _ctx) => {
    const sessionDir: string | undefined =
      typeof event.cwd === "string" ? event.cwd : undefined;

    const args = sessionDir ? [SCRIPT, sessionDir] : [SCRIPT];

    // Fire-and-forget — sync failure must never block a session
    execFile(PYTHON, args, { timeout: 30_000 }, () => {});
  });
}
```

Key decisions:

- **`execFile` not `pi.exec`** — `pi.exec` routes through the OMP shell layer. `execFile` calls the OS Python directly, avoiding shell quoting issues on Windows.
- **Fire-and-forget** — sync failure must never block a session.
- **CWD only** — no hardcoded list of repos to maintain. Each session pulls its own repo.

The Python script is intentionally minimal:

```python
# sync-repos.py
def pull(repo: Path) -> None:
    if not (repo / ".git").exists():
        return
    if not subprocess.run(["git", "remote"], cwd=repo, capture_output=True, text=True).stdout.strip():
        return
    result = subprocess.run(["git", "pull", "--ff-only"], cwd=repo, capture_output=True, text=True)
    if result.returncode != 0:
        print(f"sync-repos: pull failed in {repo}\n{(result.stderr or result.stdout).strip()}", file=sys.stderr)
```

No throttle, no stamp file — one `git pull` per session start is fast enough to not need it.

## Register

```yaml
# ~/.omp/agent/config.yml
extensions:
  - ~/repos/github.com/you/omp-sync-repos
```

## Claude Code

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [{ "type": "command", "command": "git pull --ff-only || true" }]
      }
    ]
  }
}
```

## Source

- [`ankitg12/omp-sync-repos`](https://github.com/ankitg12/omp-sync-repos)
