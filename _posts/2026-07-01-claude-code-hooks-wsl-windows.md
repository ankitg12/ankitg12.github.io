---
layout: post
title: "One ~/.claude, two environments: making Claude Code hooks work in WSL and Windows Git Bash"
date: 2026-07-01
categories: ai tools windows wsl
series: "AI coding agent productivity"
---

I run Claude Code in two environments: WSL (Ubuntu) for most work, and MSYS2/Git Bash via OMP for Windows-native sessions. The config lives in one place — `C:\Users\angaur\.claude\` — symlinked into WSL as `/root/.claude`. Same `settings.json`, same commands, same hooks.

Or so I thought.

Every hook silently failed in WSL. Retros didn't fire. Timestamps were missing. Screenshots came back black. Nothing errored loudly — the hooks just did nothing.

---

## The problem

`settings.json` hooks are shell commands. Mine looked like this:

```json
"command": "python C:/Users/angaur/.claude/plugins/cache/ankitg12/claude-code-timestamps/1.0.0/timestamps.py prompt"
```

This works in MSYS2/Git Bash: `python` resolves to the Windows Python in `PATH`, and `C:/...` is a valid path.

In WSL bash, both assumptions break:
- `python` — not found (`python3` is the binary)
- `C:/Users/...` — WSL treats this as a relative path from the current directory, resolving to `/root/C:/Users/...`, which doesn't exist

Same command, two environments, one silently wrong.

The screenshot hook had a different failure. It used `python3` from WSL to call `mss` (a screen capture library):

```bash
python3 -c "import mss, mss.tools; m=mss.MSS(); shot=m.grab(m.monitors[1]); ..."
```

WSL has no display server. `mss` grabbed a virtual screen — 1920×1080 of solid black. No error, just a useless image.

---

## The fix: two small wrapper scripts

### `pyrun.sh` — cross-env Python runner

WSL has `wslpath`, a builtin that converts `C:/...` paths to `/mnt/c/...`. MSYS2 doesn't have it, but there `python` already resolves to Windows Python and `C:/...` paths work natively.

```bash
#!/bin/bash
# Usage: pyrun.sh C:/path/to/script.py [args...]

SCRIPT="$1"
shift

if command -v wslpath &>/dev/null; then
  python3 "$(wslpath -u "$SCRIPT")" "$@"
else
  python "$SCRIPT" "$@"
fi
```

Saved to `~/tools/pyrun.sh`. Then every hook that was `python C:/path/to/script.py args` becomes:

```json
"command": "bash -c '[ -d /mnt/c ] && p=/mnt/c || p=/c; bash ${p}/Users/angaur/tools/pyrun.sh C:/path/to/script.py args'"
```

The outer `[ -d /mnt/c ]` resolves the mount prefix so the script itself can be found before `wslpath` is available to translate it. Once inside `pyrun.sh`, `wslpath` handles everything else.

### `screenshot-hook.sh` — Windows Python for captures

The screenshot hook needed Windows Python specifically — WSL Python has no display. `wslpath` again does the path translation, and we invoke Python via `pwsh.exe` which has it in scope:

```bash
#!/bin/bash
INPUT=$(cat)
PROMPT=$(python3 -c "import sys,json; d=json.loads(sys.stdin.read()); print(d.get('prompt',''))" <<< "$INPUT" 2>/dev/null)

if [[ "$PROMPT" != /screenshot* ]]; then
  exit 0
fi

if command -v wslpath &>/dev/null; then
  MNT=$(wslpath -u "C:/")
  MNT="${MNT%/}"
  PWSH=$(wslpath -u "C:/Program Files/PowerShell/7/pwsh.exe")
  mkdir -p "${MNT}/tmp"
  "$PWSH" -NoProfile -Command "python -c \"import mss,mss.tools;m=mss.MSS();s=m.grab(m.monitors[1]);mss.tools.to_png(s.rgb,s.size,output=r'C:/tmp/cc-screenshot.png')\""
else
  mkdir -p /c/tmp
  MNT=/c
  python -c "import mss,mss.tools;m=mss.MSS();s=m.grab(m.monitors[1]);mss.tools.to_png(s.rgb,s.size,output='C:/tmp/cc-screenshot.png')"
fi

echo "{\"hookSpecificOutput\":{\"hookEventName\":\"UserPromptSubmit\",\"additionalContext\":\"Screenshot pre-captured at ${MNT}/tmp/cc-screenshot.png — read it directly without running Bash.\"}}"
```

---

## The result

Same `settings.json`. Both environments. Hooks fire correctly in both.

| Hook | WSL before | WSL after | MSYS2 |
|---|---|---|---|
| Timestamps | silent fail | ✓ | ✓ |
| Retro tracker | silent fail | ✓ | ✓ |
| log_prompt.sh | silent fail (`/c/` path) | ✓ | ✓ |
| Screenshot | black 1920×1080 | ✓ (via pwsh) | ✓ |

The key insight: **hooks run in whatever bash Claude Code spawns**, and that bash differs by environment. Don't hardcode `python` or `C:/` — write one small wrapper that detects which world it's in and adapts.

---

## What I learned about `wslpath`

`wslpath` is a WSL builtin I hadn't used before. It's cleaner than string manipulation:

```bash
wslpath -u "C:/Users/angaur/tools/script.py"
# → /mnt/c/Users/angaur/tools/script.py

wslpath -w "/mnt/c/Users/angaur/tools/script.py"
# → C:\Users\angaur\tools\script.py

wslpath -m "/mnt/c/Users/angaur/tools/script.py"
# → C:/Users/angaur/tools/script.py   (forward slashes)
```

`command -v wslpath` is a reliable WSL detector — the binary exists only in WSL environments.

---

## Files

- `~/tools/pyrun.sh` — cross-env Python runner
- `~/tools/screenshot.sh` — standalone screenshot capture (for `/screenshot` skill)
- `~/tools/screenshot-hook.sh` — UserPromptSubmit hook variant with JSON output

All three live under `C:\Users\angaur\tools\` and are accessible from both environments via their respective mount paths.

---

Related posts:
- [Building a /screenshot command for Claude Code on Windows]({% post_url 2026-06-05-screenshot-claude-code-windows %})
- [claude-code-timestamps: timestamps at every CC lifecycle event]({% post_url 2026-05-29-claude-code-timestamps %})
