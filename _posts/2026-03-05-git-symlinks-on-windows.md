---
layout: post
title: "The Complete Guide to Git Symlinks on Windows: Why They Break and How to Fix Them"
date: 2026-03-05
categories: git windows symlinks
---

I spent a day debugging symlinks in a git repo on Windows. What started as "just rename a directory" turned into a deep dive through MSYS2, Python, Windows Defender, and pre-commit hooks. Here's everything I learned so you don't have to.

## The Setup

We have a repo with a `scripts/all/` directory that contains symlinks to scripts in other directories ΓÇö a flat index for tab-completion:

```plaintext
scripts/
Γö£ΓöÇΓöÇ tools/
Γöé   Γö£ΓöÇΓöÇ deploy.sh
Γöé   Γö£ΓöÇΓöÇ cleanup.sh
Γöé   ΓööΓöÇΓöÇ ...
Γö£ΓöÇΓöÇ checks/
Γöé   Γö£ΓöÇΓöÇ health.sh
Γöé   ΓööΓöÇΓöÇ ...
└── all/           ← symlinks to everything above
    Γö£ΓöÇΓöÇ deploy.sh -> ../tools/deploy.sh
    Γö£ΓöÇΓöÇ health.sh -> ../checks/health.sh
    ΓööΓöÇΓöÇ ...
```

We renamed `scripts/tools/` to `scripts/operations/`. Symlinks broke. Simple, right? Just rebuild them. That's when things got interesting.

## Problem 1: File Symlinks vs Directory Symlinks

Windows treats file and directory symlinks differently. Git bash (`ln -s`) handles file symlinks fine:

```bash
$ ln -s ../README.md LINK.md
$ ls -la LINK.md
lrwxrwxrwx 1 user 1049089 48 LINK.md -> ../README.md  # Γ£ô proper symlink
```

But directory symlinks created the same way? They silently become **copies**:

```bash
$ ln -s ../../features/myfeature myfeature
$ ls -la myfeature
drwxr-xr-x 1 user 1049089 0 myfeature  # ← directory, NOT symlink!
```

No error. No warning. Just a full copy of the directory. Git then tracks it as regular files, and your "symlinks" go stale immediately.

**Fix:** Use PowerShell `New-Item -ItemType SymbolicLink` for directory symlinks:

```powershell
New-Item -ItemType SymbolicLink -Path "myfeature" -Target "..\..\features\myfeature"
```

Or Python's `os.symlink` with `target_is_directory=True`:

```python
os.symlink("../../features/myfeature", "myfeature", target_is_directory=True)
```

## Problem 2: Python os.path.exists() Lies About Symlinks

After creating file symlinks with `ln -s` in git bash, they worked perfectly in bash:

```bash
$ cat scripts/all/deploy.sh  # Γ£ô reads through symlink fine
$ readlink scripts/all/deploy.sh
../operations/deploy.sh  # Γ£ô correct target
```

But Python disagreed:

```python
>>> os.path.islink("scripts/all/deploy.sh")
True   # Γ£ô yes, it's a symlink
>>> os.path.exists("scripts/all/deploy.sh")
False  # Γ£ù but it "doesn't exist"?!
>>> os.stat("scripts/all/deploy.sh")
WinError 123: The filename, directory name, or volume label syntax is incorrect
```

The symlink target was `../operations/deploy.sh` (forward slashes). Windows `stat()` can't resolve relative paths with forward slashes through symlinks. The file is there, bash can read it, but Python's Windows implementation chokes.

**Root cause:** MSYS2's `ln -s` creates Windows symlinks with forward-slash targets. Windows APIs need backslashes to resolve relative symlinks.

## Problem 3: MSYS=winsymlinks:nativestrict

Setting `MSYS=winsymlinks:nativestrict` before `ln -s` creates proper native Windows symlinks that Python can resolve:

```bash
$ MSYS=winsymlinks:nativestrict ln -sf "../operations/deploy.sh" "scripts/all/deploy.sh"
$ python3 -c "import os; print(os.path.exists('scripts/all/deploy.sh'))"
True  # Γ£ô finally!
```

But this has a performance problem...

## Problem 4: Windows Defender Makes ln -s 100x Slower

With `MSYS=winsymlinks:nativestrict`, each `ln -s` call:
1. Spawns a new process
2. Creates a native Windows symlink
3. Triggers Windows Defender real-time scanning

For 40+ symlinks in a loop:

```bash
# This takes 120+ seconds and times out
for f in scripts/operations/*.sh; do
    ln -sf "../operations/$(basename $f)" "scripts/all/$(basename $f)"
done
```

The same operation in Python takes 0.1 seconds:

```python
# 40+ symlinks in 0.1 seconds
for script in glob.glob("scripts/operations/*.sh"):
    name = os.path.basename(script)
    os.symlink(os.path.join("..", "operations", name),
               os.path.join("scripts/all", name))
```

Python makes direct syscalls from a single process. No process spawn per symlink, no repeated AV scanning.

## Problem 5: pre-commit check-symlinks Hook

The [pre-commit](https://pre-commit.com/) framework has a `check-symlinks` hook. Its implementation is dead simple:

```python
# From pre-commit-hooks/check_symlinks.py
if os.path.islink(filename) and not os.path.exists(filename):
    print(f'{filename}: Broken symlink')
```

This combines problems 2 and 3: if your symlinks were created by bash `ln -s` (forward-slash targets), Python reports them as broken even though they work perfectly in bash.

## Problem 6: Git Normalizes Symlink Targets

Git always stores symlink targets with forward slashes, regardless of what's on disk:

```bash
$ git cat-file -p :scripts/all/deploy.sh
../operations/deploy.sh  # forward slashes in git
```

Even if you create a symlink with backslashes on disk, git stores forward slashes. On checkout, git recreates the symlink ΓÇö and depending on your `core.symlinks` setting and MSYS configuration, you might get forward or backslash targets.

## The Solution

After trying every combination, here's what actually works:

### 1. Use Python for symlink creation

```python
#!/usr/bin/env python3
"""Cross-platform symlink builder. Works on Linux and Windows."""
import os, glob

all_dir = "scripts/all"
for entry in sorted(os.listdir("scripts")):
    subdir = os.path.join("scripts", entry)
    if not os.path.isdir(subdir) or entry in {"all", "lib"}:
        continue
    for script in glob.glob(os.path.join(subdir, "*.sh")):
        name = os.path.basename(script)
        # os.path.join uses OS-native separator automatically
        target = os.path.join("..", entry, name)
        link = os.path.join(all_dir, name)
        os.symlink(target, link)
```

`os.path.join` produces backslashes on Windows, forward slashes on Linux. `os.symlink` creates native symlinks on both. `os.path.exists` resolves them correctly on both.

### 2. Use `.editorconfig` and `.gitattributes`

```ini
# .editorconfig
[*]
end_of_line = lf
charset = utf-8
```

```conf
# .gitattributes
* text=auto eol=lf
```

### 3. Use pre-commit for enforcement

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: check-symlinks
      - id: mixed-line-ending
        args: [--fix=lf]
      - id: end-of-file-fixer
```

### 4. Set `core.symlinks = true` in git config

```bash
git config core.symlinks true
```

## TL;DR Cheat Sheet

| What | Linux | Windows |
|------|-------|---------|
| Create file symlink | `ln -s target link` | `ln -s target link` (works in git bash) |
| Create dir symlink | `ln -s target link` | `New-Item -ItemType SymbolicLink` or `os.symlink(..., target_is_directory=True)` |
| Bulk symlinks | `ln -s` in loop | Python `os.symlink` (100x faster than bash loop) |
| Verify symlink works | `os.path.exists()` | Only if created with backslash targets |
| Force native symlinks | N/A | `MSYS=winsymlinks:nativestrict` before `ln -s` |
| pre-commit compatibility | Just works | Use Python `os.symlink` with `os.path.join` for native separators |

## Key Takeaway

Symlinks on Windows are not broken ΓÇö they're just picky about separators. Use Python's `os.symlink` with `os.path.join` for cross-platform compatibility, and avoid bash `ln -s` in loops on Windows (AV scanning makes it 100x slower).

The deeper lesson: when something "works in bash but not in Python" on Windows, it's almost always a forward-slash vs backslash issue in path resolution.
