---
layout: post
title: "Your Agent Started a Background Process. Where Did It Go?"
date: 2026-08-14
categories: ai agents productivity terminal
series: "AI coding agent productivity"
---

My coding agent printed one line — `Launch start: boot-watch` — and then nothing.
Somewhere a process was tailing a serial console with a 900-second timeout, and my
only evidence of it was that banner. I could not see its output, could not tell
whether it was healthy, and when I cancelled it in the UI, two processes survived.

The daemon registry told the real story. Across sessions: 269 launch records, 226
exited, **37 failed**, 5 still ready. Names like `pd-test2`, `omp-hold-test`,
`livemon-demo3`. A graveyard of things I started and never watched.

The tempting fix is a better log viewer. The correct fix is narrower: **if a
long-running process is not visible, it does not run.**

## Step one: make the opaque path unavailable

Most agent harnesses auto-approve tool tiers. Mine defaulted to approving
`exec`, so the background-launch tool fired without a prompt — working as
configured, and configured wrong.

I considered an approval prompt and rejected it. Approval was per-*tool*, not
per-*operation*, so it would also prompt on messaging and log reads, and the
tool exposed no formatter for launch specs — the prompt would not even have
shown me *what* was about to run. Approving a thing you cannot see is theatre.

So: disable background daemons outright, and intercept the call in a
pre-execution hook that re-routes it into a terminal pane I can watch. The hook
fires *before* the tool runs, so both can be true at once — the opaque path
stays disabled, and the visible path still works.

## Step two: don't write the terminal layer

I wrote backends for two multiplexers before searching for prior art. That was
the wrong order, and it cost more than it saved.

[`pi-terminal-mux`](https://www.npmjs.com/package/pi-terminal-mux) already
abstracts herdr/wezterm/tmux/zellij and friends behind one surface API, with a
headless fallback. Its README states the case plainly: any extension needing
terminal interaction should depend on it rather than re-implement backend
detection. I deleted ~190 lines of my own and kept only the part it genuinely
does not model — refusing launch semantics I cannot honour, like restart
policies and port-based readiness, instead of silently downgrading them into a
success-shaped result.

## The bug worth the whole post: LF is not Enter

The WezTerm path created its pane, sent the command, and hung. The pane showed:

```
PS C:\Users\...>
>> & 'C:\Users\...\launch.ps1'
```

That `>>` is PowerShell's *continuation* prompt. The command had been typed but
never submitted. The library sends `command + "\n"` — and on a Windows ConPTY, a
bare line feed does not submit a line. Carriage return does.

One character of proof, on the already-parked pane:

```bash
printf '\r' | wezterm cli send-text --pane-id 7 --no-paste
# -> tick 1 / tick 2 / tick 3 / DONE_MARK
```

This is the terminal's [line
discipline](https://en.wikipedia.org/wiki/Line_discipline) — the kernel layer
between raw bytes and "the shell got a line." On Unix, `ICRNL` translates CR to
NL on input, which is why `"\n"` appears to work everywhere and quietly does not
here. If you drive terminals by pasting text, this will eventually bite you:

| You send | Unix pty (ICRNL) | Windows ConPTY |
|---|---|---|
| `\n` | submits | continuation prompt, hangs |
| `\r` | submits | submits |

The failure mode is the dangerous kind: the send call returns success, the pane
looks busy, and nothing ran.

## Reading the pane back is a separate problem

Making a process visible to a *human* is not the same as making it legible to
the *agent*. The pane rendered perfectly on screen; the programmatic capture
came back garbled:

```
PS ...> 1..3 | % { "pr
obe $_" }; "PROBE
PS ...> 1..3 | % { "pr
obe $_" }; "PROBE" +
```

Those are PSReadLine's per-character redraws of a line that wrapped in a narrow
split. Any readiness check regexing that text is guessing.

Two fixes, and I took the second:

1. **Un-wrap the capture.** The herdr backend already supports a
   `recent_unwrapped` read source — but the library's unified `readScreen()`
   hardcodes `recent` and offers no way to select it.
2. **Never read the echo at all.** Have the launched script print a sentinel as
   its first act, and slice the capture after its last occurrence. Width-
   independent, no heuristics, no un-wrapping.

```ts
export const OUTPUT_SENTINEL = "<<<VISIBLE-LAUNCH>>>";

export function readClean(surface: string, lines = 60): string {
  const screen = readScreen(surface, lines);
  const at = screen.lastIndexOf(OUTPUT_SENTINEL);
  if (at === -1) return screen;
  return screen.slice(at + OUTPUT_SENTINEL.length).replace(/^\r?\n/, "");
}
```

A related trick: write the command to a script file and send `& 'path.ps1'`
instead of a 200-character one-liner. The echoed line stays short, so it cannot
wrap in the first place. The library has exactly this helper — and it hardcodes
`#!/bin/bash`, so PowerShell users must rebuild it.

## Don't replace one graveyard with another

The first version of that script-file trick called `mkdtemp()` per launch. Ten
minutes of testing left 23 orphaned directories — I had rebuilt the exact
accumulation problem I set out to remove, one layer down.

The naive fix, one file per launch *name*, overwritten, introduces a race: the
pane reads the script *after* the launch call returns, so a second launch can
rewrite a body the first shell has not read yet. The shape that actually works
is **unique files in one shared directory, pruned by age**:

```ts
export function writeLaunchScript(name: string, body: string): string {
  mkdirSync(SCRIPT_DIR, { recursive: true });
  pruneOldScripts();                       // anything older than an hour
  const path = join(SCRIPT_DIR, `${safe(name)}-${unique()}.ps1`);
  writeFileSync(path, `${body}\r\n`, "utf8");
  return path;
}
```

Unique so live launches are immutable; shared and pruned so the directory stays
bounded. The test asserts both: distinct paths for same-named launches, and that
pruning removes stale files while leaving live ones alone.

Same discipline for the test harness. A live test that leaks a pane on failure is
the same bug wearing a lab coat — and `finally` does not save you, because a
runner that hard-kills at 30 seconds never reaches it. Put the test's own
deadline *inside* the runner's timeout, and add a signal handler for the kill
case.

## What this actually buys

Nothing exotic. A process starts in a pane I can watch, in a split that keeps the
full terminal width so output does not wrap. The agent polls a clean read instead
of a smeared one. Launch semantics the surface cannot honour are refused rather
than quietly dropped.

The general rule I would keep even if I threw away every line of this code:

> A tool result that *reports* success while the surface shows a `>>` prompt is
> worse than a failure. Assert on the surface, never on the return value.

Three of these findings were library bugs rather than mine, so they went upstream
with the raw pane output attached. That took ten minutes and is the only part of
this work that helps anyone else.

## Update, same evening: two more defects, and one design principle

I went back to the tool expecting to confirm it worked. It did not, twice, and
the second failure is more interesting than the first.

### The launcher assumed it owned an empty prompt

A launch failed with PowerShell's parser complaining:

```
>> :/prompt& 'C:\...\vl-sentinel2-mst53fsjc1b31f1d.ps1'
The ampersand (&) character is not allowed.
```

The pane had half-typed text sitting on its input line. The launcher typed its
own invocation onto the end of it, producing `:/prompt& '...'`. No line-clear
existed anywhere in the send path — the tool worked only when the prompt
happened to be empty, an unstated precondition rather than a broken tool.

The fix is one call, but *which* key matters, and I got it wrong first. My
instinct was Ctrl-U. Asking the machine instead of my memory:

```powershell
Get-PSReadLineKeyHandler -Bound |
  Where-Object { $_.Key -match 'Ctrl\+u|Escape' } |
  ForEach-Object { "$($_.Key) => $($_.Function)" }
# Escape => RevertLine
```

`Escape => RevertLine`, and **no Ctrl+u binding at all**. Ctrl-U would have been
a silent no-op: a clear that clears nothing, hiding the bug rather than fixing
it. Two further details decided the shape of the patch — the mux library already
exports a raw `sendEscape()` (the `sendCommand()` primitive appends a
terminator and would have *executed* the escape), and the clear must precede
the command **text**, not the carriage return, because those are emitted by
different code paths and clearing after the text erases the command you just
typed.

### A tool that intercepts `start` owns the whole lifecycle

The second defect is the one worth generalising. The extension intercepts
`start` and re-routes it into a pane. Every *other* lifecycle verb —
`logs`, `stop`, `ps`, `wait`, `restart` — fell through to the untouched
background-launch tool, which had never heard of the process:

```
Unknown daemon xlsx-read-test. Available: afk-engage, agent-hub-post-commit, ...
```

The process was running perfectly. Only the *questions about it* failed. I hit
this three times in one evening before recognising it as a design error rather
than three unrelated annoyances.

> If you intercept the verb that creates a resource, you have implicitly taken
> ownership of every verb that observes or destroys it. Compensating in banner
> prose — "read the pane with `readClean(...)`" — is documentation where an API
> contract was required.

Before writing the shim I checked whether the harness could simply be told about
the process. It cannot: the daemon broker keeps its registry in a private field
and exports no adopt/attach API, and a code search across the public extension
ecosystem turned up no prior art for intercepting these verbs. The multiplexer
library, though, already supplies every primitive needed — surface creation,
screen reads, close, and split management. The shim is a `Map` from launch name
to surface id plus a switch, and nothing more.

That registry is the part with teeth. Two rules fell out of it:

| Verb | Answer | Why |
|---|---|---|
| `logs` | clean read of the surface | the pane *is* the log |
| `stop` | close surface, delete entry | otherwise the id is unreachable forever |
| `wait` | poll for an exit sentinel | needed a new marker — see below |
| `restart` | close, relaunch from retained spec | the spec must be kept, not just the id |
| `ps` | list the registry, **and say so** | it is not a merged view |
| anything unowned | pass through untouched | real daemons must keep working |

`wait` forced a change to the launched script. Previously it printed one
sentinel *before* the command; honest waiting needs one *after* it, carrying the
exit code:

```powershell
Write-Output ("<<<VISIBLE-LAUNCH-EXIT:" +
  $(if ($null -eq $LASTEXITCODE) { if ($?) { 0 } else { 1 } } else { $LASTEXITCODE }) +
  ">>>")
```

`$LASTEXITCODE` is set only by native executables, so a failing cmdlet needs the
`$?` fallback. And on timeout, `wait` reports *"still running, not stopped"* —
a `wait` that silently returns success when it simply gave up is worse than
having no `wait` at all.

Two smaller traps, both caught by tests rather than by thinking. A duplicate
`start` name originally overwrote its map entry, stranding the previous pane
with nothing holding its id — unreachable by any verb, the graveyard problem
again, one layer up; it now refuses, as the real launcher does. And `ps` first
printed the last line of the generated wrapper script, which is the exit marker
and tells the reader nothing; it now stores the quoted invocation the caller
actually asked for.

### On honest scope

The `ps` answer was the hardest call. This shim cannot read the harness's own
daemon table, so any `ps` it answers is partial by construction. Blocking an
unscoped query with a subset silently redefines what `ps` means. The compromise:
answer only when the shim owns something, and state plainly in the output that
it is one registry and not a merged list. A partial answer that admits it is
partial is usable; one that does not is a lie with good formatting.


## Source

- [pi-terminal-mux on npm](https://www.npmjs.com/package/pi-terminal-mux) — the multiplexer abstraction
- [maplezzk/pi-extensions](https://github.com/maplezzk/pi-extensions) — its repository
- [Issue #90](https://github.com/maplezzk/pi-extensions/issues/90) — the three findings above, filed upstream
- [TTY architecture: pty, line discipline, shell, terminal](https://terminfo.dev/fundamentals/tty-architecture) — the clearest explanation of the layer that ate my LF
- [Line discipline (Wikipedia)](https://en.wikipedia.org/wiki/Line_discipline)
- [WezTerm CLI reference](https://wezterm.org/cli/cli/index.html)
- [`Get-PSReadLineKeyHandler`](https://learn.microsoft.com/en-us/powershell/module/psreadline/get-psreadlinekeyhandler) — ask the shell which keys are bound instead of guessing
