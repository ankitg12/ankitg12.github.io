---
layout: post
title: "Fixing a Windows→WSL shim for multi-line scripts (and a Python detour I shouldn't have taken)"
date: 2026-07-16
categories: windows wsl tools
series: "AI coding agent productivity"
---

`wu` is a one-line `.cmd` shim I use on Windows to run things inside WSL without typing `wsl -d Ubuntu-26.04 --` every time:

```bat
@echo off
wsl -d Ubuntu-26.04 -- bash -lc %*
```

It works fine for `wu "cd ~/repo && ls"`. It falls apart the moment the payload is a multi-line script — a heredoc, or a file piped in with nested quotes and `$` interpolation. `%*` round-trips the whole thing through cmd.exe's argument expansion, which mangles line breaks and quoting before bash ever sees it. The known workaround was: write the script to a Windows scratch path, `cp` it into WSL, run it, delete it. Three steps for something that should be one.

## First instinct: reach for Python

My first move was to rewrite the shim as a Python script that would base64-encode the payload, or write it to a WSL temp file via `mktemp` and `cat`, then execute that file with `bash`. I got partway into building this — argument parsing, a `-f/--file` flag, a two-step "write then exec" subprocess dance — before testing it against a real multi-line payload.

It didn't work, and debugging it burned time chasing the wrong thing. `mktemp` inside a `bash -c '...'` invocation kept returning empty when I called it through my coding agent's `bash` tool, which made me suspect a WSL interop quirk with stdin redirection. It wasn't WSL. It was the tool: that particular `bash` tool pre-expands literal `$NAME` patterns in the command *string* itself, before any real shell sees it — even inside single quotes. `$f` and `$y` were silently evaluating to empty strings inside the harness's own preprocessing, not inside bash. Switching to a raw subprocess call (bypassing the tool's string substitution) immediately showed `mktemp` and command substitution working exactly as expected.

Once that noise was out of the way, the two-file approach *did* work — but I'd built a whole Python wrapper (argument parsing, temp-file plumbing, cleanup logic) to solve a problem that already has a built-in one-line solution.

## The actual fix: `bash -s`

`bash` has had first-class support for "read the whole script from stdin" since forever: `bash -s` (or `bash -l -s` for a login shell). No temp file, no round-trip, no quoting through cmd.exe at all — the script bytes go straight to bash's stdin exactly as sent.

```
$ cat script.sh | wsl -d Ubuntu-26.04 -- bash -l -s
```

That's the entire trick. The shim just needs to pick this path when it's *not* given a command-line argument:

```bat
@echo off
if "%~1"=="" (
    wsl -d Ubuntu-26.04 -- bash -l -s
) else (
    wsl -d Ubuntu-26.04 -- bash -lc %*
)
```

`wu "cmd"` keeps working exactly as before. `wu <<'EOF' ... EOF` or `cat script.sh | wu` now streams the whole script through stdin, untouched:

```
$ cat <<'EOF' | wu
cd ~/repo
cat <<'INNER' > foo.txt
literal $HOME and "quotes" and `backticks` survive here
INNER
ls -la
EOF
```

Every `$`, backtick, and quote in that payload comes out the other side exactly as typed, because it's never been command-line text — it's just bytes on a pipe.

## The lesson

I skipped a "does something already do this" check before writing code, and paid for it twice: once in wasted implementation effort, once in wasted debugging effort chasing a false lead (WSL stdin quirk) caused entirely by my own test harness's string substitution, not by the thing I was actually building. The fix ended up being four lines in an existing file with zero new dependencies, because `bash -s` was already the right tool — I just hadn't asked whether it existed before reaching for a new one.

## Source

- Shim: `~/.local/bin/wu.cmd` (personal dotfiles, not public)
- `bash(1)` — `-s` flag: "If there are no operands, `-s` causes bash to read commands from standard input."
