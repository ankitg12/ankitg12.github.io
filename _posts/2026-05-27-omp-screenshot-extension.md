---
layout: post
title: "Building a /screenshot command for OMP — one option away from working"
date: 2026-05-27
categories: ai tools windows
---

I wanted a `/screenshot` command in OMP that captures the screen and attaches the image to the conversation. "Should be easy," I thought. "Just one PowerShell command."

It was not easy. Here's what actually happened.

---

## The idea

OMP (Oh My Pi) lets you register slash commands via extensions. The plan:

1. `/screenshot` fires
2. Captures the full screen
3. Prepends `@/path/to/ss.png` to the editor so you add your question and send

Extensions are two files — a `.ts` and a `package.json`. OMP compiles the TypeScript at load time. No build step.

---

## Attempt 1 — inline PowerShell via `execSync`

```typescript
execSync(`pwsh -NoProfile -Command "${ps.replace(/"/g, '\\"').replace(/\n/g, '; ')}"`, {
  timeout: 10_000,
});
```

Result: `spawnSync C:\WINDOWS\system32\cmd.exe ETIMEDOUT`.

Node uses `cmd.exe` to run shell commands. The inline PowerShell string mangled quoting. The command never ran — it just timed out.

Lesson: **never pass complex PowerShell via inline `-Command`**. Write a `.ps1` file and use `-File`.

---

## Attempt 2 — PowerShell via `.ps1` file

```typescript
writeFileSync(ps1Path, script, "utf8");
execSync(`pwsh -NoProfile -File "${ps1Path}"`, { timeout: 15_000 });
```

Still ETIMEDOUT. PowerShell startup on this machine takes ~13 seconds cold. 15s wasn't enough once Node's overhead was added. Also `pwsh` wasn't resolving from OMP's spawn environment.

Lesson: **never assume shell tools are in PATH from a spawned process**. Use the full executable path.

---

## Attempt 3 — switch to Python + PIL

PowerShell startup is the real problem. Python's `PIL.ImageGrab` does the same thing in 0.1s:

```python
from PIL import ImageGrab
ImageGrab.grab().save(r'C:\tmp\ss.png')
```

Tested from bash: 3.3s total (mostly Python startup), 0.1s actual capture. Switched the extension:

```typescript
const result = spawnSync(PYTHON, [
  "-c",
  `from PIL import ImageGrab; ImageGrab.grab().save(r'${out}')`,
], { timeout: 10_000, windowsHide: true });
```

Result in RPC test session: **✅ 0.65s, success.**  
Result in interactive TUI session: **❌ ETIMEDOUT, status=null, 78ms.**

---

## The real problem — PTY inheritance

The failure pattern was suspicious: `status=null`, ETIMEDOUT, but only 78ms after spawning. That's not a timeout — that's an instant kill.

The consistent difference:

| Mode | Result |
|------|--------|
| RPC (headless) | ✅ every time |
| Interactive TUI | ❌ every time |

OMP in interactive mode owns a PTY (pseudo-terminal). `spawnSync` without explicit `stdio` inherits the parent's file descriptors — including OMP's terminal. The child process starts, inherits OMP's PTY, and gets killed immediately.

The fix was one option:

```typescript
], { timeout: 30_000, windowsHide: true, stdio: 'pipe' });
```

`stdio: 'pipe'` isolates the child process from OMP's terminal entirely. Python spawns clean, captures the screen, exits. Done.

---

## What the debug logger revealed

Without a log file, this would have taken much longer. Adding one line per operation:

```typescript
const LOG = "C:\\tmp\\omp-screenshots\\omp-screenshot.log";
function log(msg: string) {
  try { appendFileSync(LOG, `${new Date().toISOString()} ${msg}\n`); } catch {}
}
```

The log showed the exact failure:

```
10:45:07.980 spawning python: C:\...\python.exe
10:45:08.059 status=null error=SystemError: spawnSync ... ETIMEDOUT stdout= stderr=
```

78ms. `status=null`. That's not a timeout — that's PTY inheritance killing the child. The log made the diagnosis instant.

---

## Final extension

```typescript
const result = spawnSync(PYTHON, [
  "-c",
  `from PIL import ImageGrab; ImageGrab.grab().save(r'${out}')`,
], { timeout: 30_000, windowsHide: true, stdio: 'pipe' });
```

Full source: [github.com/ankitg12/omp-screenshot](https://github.com/ankitg12/omp-screenshot)

---

## Lessons

| What I assumed | What was true |
|----------------|---------------|
| PowerShell is fine for a quick screen capture | 13s cold start. Use PIL. |
| `pwsh` is in PATH | Not in OMP's spawn environment. Use full path. |
| 10s timeout is enough | Not with PTY inheritance killing the child in 78ms |
| RPC test = proof it works | RPC is headless. Interactive TUI behaves differently. |
| `status=null` means timeout | It means the process was killed — PTY, signal, or spawn failure |
| More timeout fixes ETIMEDOUT | `stdio: 'pipe'` fixes it. Timeout is irrelevant when the child dies in 78ms. |

The working extension is 52 lines. The journey to get there was not.

---

## References

- [github.com/ankitg12/omp-screenshot](https://github.com/ankitg12/omp-screenshot)
- [github.com/can1357/oh-my-pi](https://github.com/can1357/oh-my-pi) — OMP extension API
- [PIL ImageGrab docs](https://pillow.readthedocs.io/en/stable/reference/ImageGrab.html)
- [Node.js spawnSync — stdio option](https://nodejs.org/api/child_process.html#child_processspawnsynccommand-args-options)
