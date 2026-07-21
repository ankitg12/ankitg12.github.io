---
layout: post
title: "Running commands in WSL from Windows: stop fighting the quoting"
date: 2026-07-21
categories: wsl windows shell productivity
---

You have a Windows shell open and you want to run one Linux command in WSL. You type what looks obviously correct:

```
wu 'cd ~/repo && git show SHA:path/file | grep "one|two"'
```

…and WSL greets you with `rtr: command not found` or `unexpected EOF while looking for matching quote`. The command is fine. The *quoting* died three shells ago.

> An [earlier post]({% post_url 2026-07-16-wu-cmd-wsl-multiline-fix %}) covered the *mechanical* fix for multi-line scripts in this same `wu` wrapper. This one is about the underlying *why* — the layered-quoting model that explains the whole class of failures, and the one principle that dissolves it.

## Why this breaks

When you invoke a Linux command from a Windows wrapper, the string you typed is re-parsed by **three** different parsers before a single byte reaches your program:

```mermaid
graph LR
  A[MSYS / git-bash] --> B[cmd.exe]
  B --> C[wsl.exe arg split]
  C --> D[bash -lc]
```

Each layer has its own idea of what a quote and a pipe mean:

| Layer | What it does to your string |
|---|---|
| MSYS / git-bash | strips one layer of quotes, expands `$vars`, splits on its rules |
| `cmd.exe` | does **not** treat `'single quotes'` as quotes at all; sees bare `|` as a *cmd pipe* |
| `wsl.exe` | re-splits the remaining args on spaces |
| `bash -lc "$@"` | finally parses what's left as a shell command |

The classic failure: `grep "one|two"` — `cmd.exe` sees the `|` **outside** any quoting it recognizes (it doesn't honor single quotes), interprets it as a pipe, and tries to run `two` as a separate program. Hence `two: command not found`. Your carefully-quoted regex never survived layer two.

This is not a bug in any one tool — it is the unavoidable result of stacking parsers with incompatible quoting rules. Microsoft's own WSL issue tracker has years of threads on it ([#3284](https://github.com/microsoft/WSL/issues/3284), [#11635](https://github.com/microsoft/WSL/issues/11635), [#1625](https://github.com/microsoft/WSL/issues/1625)). There is no combination of escaping that is robust across all of them.

## The fix: send the command as bytes, not as arguments

Arguments get parsed. **stdin does not.** If you pipe your command into `bash` over stdin, none of the intermediate layers get to tokenize it — they only see an opaque byte stream. bash reads it as a script and runs it verbatim.

A tiny wrapper makes this ergonomic. Here is the whole thing (`wu.cmd`):

```bat
@echo off
if not defined WU_TIMEOUT set WU_TIMEOUT=180
if "%~1"=="" (
    rem no args: read the command from stdin
    wsl -- timeout -k 5 %WU_TIMEOUT% bash -l -s
) else (
    rem args: run them directly (fine for SIMPLE commands only)
    wsl -- timeout -k 5 %WU_TIMEOUT% bash -lc %*
)
```

Two modes fall out of this:

**Simple command → arg mode, double quotes:**

```
wu "echo hi; whoami"
```

**Anything with pipes, quotes, `$vars`, or `colon:paths` → stdin mode:**

```bash
printf '%s\n' 'cd ~/repo && git show SHA:path | grep "one|two"' | wu
```

The `printf '%s\n' '…'` hands one literal line to `wu`'s stdin. `bash -l -s` reads it as a script. The `|`, the `"quotes"`, the `$vars` all survive because nothing between you and bash was allowed to parse them.

Heredocs work too, which is lovely for multi-line scripts:

```bash
wu <<'EOF'
cd ~/repo
for f in *.log; do
  grep -c ERROR "$f" | sed "s/^/$f: /"
done
EOF
```

The `'EOF'` (quoted) means even `$f` passes through untouched — the remote bash expands it, not your local shell.

## The rule of thumb

- **Reaching for a pipe, a nested quote, a `$var`, or a `path:with:colons`?** Use stdin: `printf '%s\n' '…' | wu`.
- **Just a plain command or two?** `wu "cmd; cmd"` is fine.

Once you internalize *"arguments get re-parsed, stdin doesn't,"* the whole class of Windows↔WSL quoting bugs disappears. The same principle applies well beyond WSL: any time you're shipping a command across a boundary that re-tokenizes (SSH, `docker exec`, `kubectl exec`, CI YAML), prefer a stdin/heredoc channel over cramming it into arguments.

## Source

- Microsoft WSL quoting threads: [#3284](https://github.com/microsoft/WSL/issues/3284), [#11635](https://github.com/microsoft/WSL/issues/11635), [#1625](https://github.com/microsoft/WSL/issues/1625)
- [A Guide to Invoking WSL](https://devblogs.microsoft.com/commandline/a-guide-to-invoking-wsl/) — Microsoft Command Line blog
- `bash -s` reads the script from stdin; `bash -lc` parses its argument — the whole trick is choosing the former for anything non-trivial.
