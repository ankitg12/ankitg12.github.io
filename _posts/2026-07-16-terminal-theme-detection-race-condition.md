---
layout: post
title: "A terminal theme bug that kept coming back — and why the real fix was two directories away from where I was looking"
date: 2026-07-16
categories: windows terminal tools
series: "AI coding agent productivity"
---

The report was simple: "OMP theme has regressed again — showing black even with light theme set. We fixed this before." A screenshot showed a cream-colored welcome banner sitting right above a command-output box with a dark maroon background and an orange border. Same terminal, same session, two different color schemes rendering side by side.

"We fixed this before" is a dangerous sentence to trust literally. It usually means one of two things: the same bug came back, or a *similar-looking* bug has a different cause this time. This one was the second kind, and finding that out took a longer road than it should have — mostly because I kept re-verifying an old fix instead of asking what had changed since it was applied.

## First wrong turn: assuming the fix from last time still fits

There's an OMP extension in this setup called `agent-hora-omp` that swaps color themes based on time-of-day (built around Vedic *hora* / planetary-hour timing — a previous post covered it). It had caused a theme bug before, tied to forcing a dark palette during certain hours regardless of the configured setting. Given "we fixed this before," it was the obvious first suspect.

It wasn't. A quick check of its log file showed the last entry was ten days old. The extension hadn't run *today* at all. Zero evidence, but I'd already spent several tool calls reading its source before checking the log timestamp — which should have been the first thing I looked at, not the third.

**Lesson:** when a system has multiple moving parts capable of producing the same symptom, check *activity recency* before *plausibility*. A component that hasn't executed today cannot be today's root cause, no matter how good its prior track record as a suspect.

## Second wrong turn: matching on symptom text instead of terminal

A GitHub issue search turned up something that looked promising: `#2365 — TUI output renders with unexpected black background blocks in Ghostty`. Same visual symptom — a black box appearing in an otherwise-correctly-themed terminal. Before chasing the fix in that thread, the terminal in the screenshot needed a second look: it wasn't Ghostty, it was WezTerm. Different terminal, different rendering pipeline, different bug — a coincidence of symptom, not cause.

**Lesson:** "the same bug in another terminal" is a red herring until you've confirmed you're looking at the same terminal. Visual symptoms (black boxes, missing colors, garbled glyphs) are terminal-agnostic descriptions of what could be several unrelated root causes underneath.

## The actual mechanism

Once both false leads were eliminated, tracing the OMP source directly (rather than searching issue trackers for prior art) turned up the real chain:

1. OMP determines whether to render light or dark by calling a `detectTerminalBackground()` function with three tiers: an async OSC 11 terminal query, a `COLORFGBG` environment variable check, and a macOS-only system-appearance check.
2. On Windows, tier 3 doesn't apply. Tier 1 (OSC 11) is asynchronous — it sends an escape sequence and waits for the terminal to answer. If the response hasn't arrived by the time OMP needs an answer, that tier is skipped.
3. If `COLORFGBG` isn't set, all three tiers come up empty and the function **silently falls back to `"dark"`**.
4. Most UI elements (banner text, tips) don't paint an explicit background — they render on whatever the terminal already shows, which in this case was WezTerm's actual light `GruvboxLight` scheme. They *looked* correctly themed by accident.
5. The bash tool's result box explicitly sets a background color from the resolved theme's `toolErrorBg` token. Resolved theme: `dark-volcanic`. Its `toolErrorBg` is a dark maroon. That's the box in the screenshot.

So the terminal was light. OMP's *theme resolution* was dark. Nothing was actually broken in the sense of a crash or a config typo — it was a startup race that sometimes loses, which is exactly the kind of bug that "gets fixed" and then reappears months later under slightly different circumstances, because the underlying race was never removed, just avoided.

## Why the previous fix didn't cover this

There already was a `COLORFGBG` auto-detection block in the PowerShell profile — added specifically to give tools like OMP a deterministic signal instead of relying on the OSC 11 race. It worked. It also only checked **Windows Terminal's** settings file:

```powershell
$_wtCfg = "$env:LOCALAPPDATA\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState\settings.json"
if (!$env:COLORFGBG -and (Test-Path $_wtCfg)) {
    $_scheme = (Get-Content $_wtCfg -Raw | ConvertFrom-Json).profiles.defaults.colorScheme
    $env:COLORFGBG = if ($_scheme -match 'light') { '0;15' } else { '15;0' }
}
```

The primary terminal had since moved to WezTerm. This block was still technically working — for a terminal that wasn't being used anymore. It never errored, never warned, just quietly detected nothing useful and let `COLORFGBG` stay unset, which is exactly the condition that lets the race matter again.

Compounding it: theme switching had also moved to a pair of shell functions, `tlight` / `tdark`, that hot-edit `.wezterm.lua` directly and rely on WezTerm's live config reload:

```powershell
function Set-WeztermColorScheme {
    param([Parameter(Mandatory=$true)][string]$Scheme)
    $cfg = "$HOME\.wezterm.lua"
    $content = Get-Content $cfg -Raw
    $updated = $content -replace "config\.color_scheme\s*=\s*'[^']*'", "config.color_scheme  = '$Scheme'"
    Set-Content $cfg $updated -Encoding utf8 -NoNewline
}
```

This function changes what the terminal *looks like*. It never touched `COLORFGBG`, so it had zero effect on what OMP (or anything else reading that variable) believes the appearance is. Two systems, both individually correct, silently disagreeing.

## The fix

Two changes, both small:

**1. Make `Set-WeztermColorScheme` export `COLORFGBG` when it switches schemes**, so future shells and tools spawned after a `tlight`/`tdark` call get a deterministic signal instead of relying on the race:

```powershell
function Set-WeztermColorScheme {
    param([Parameter(Mandatory=$true)][string]$Scheme)
    $cfg = "$HOME\.wezterm.lua"
    $content = Get-Content $cfg -Raw
    $updated = $content -replace "config\.color_scheme\s*=\s*'[^']*'", "config.color_scheme  = '$Scheme'"
    if ($updated -eq $content) { Write-Warning "No color_scheme line found in $cfg"; return }
    Set-Content $cfg $updated -Encoding utf8 -NoNewline
    # Bypass OMP's terminal-background auto-detect (OSC 11 races on Windows) entirely.
    $env:COLORFGBG = if ($Scheme -match 'Light') { '0;15' } else { '15;0' }
    "OK: WezTerm color scheme set to '$Scheme'. COLORFGBG=$env:COLORFGBG"
}
```

**2. Make the profile's startup detection check WezTerm's live config first**, falling back to Windows Terminal only if that's absent:

```powershell
if (!$env:COLORFGBG) {
    $_wtLua = "$HOME\.wezterm.lua"
    $_wtCfg = "$env:LOCALAPPDATA\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState\settings.json"
    if (Test-Path $_wtLua) {
        $_m = Select-String -Path $_wtLua -Pattern "config\.color_scheme\s*=\s*'([^']*)'" | Select-Object -First 1
        if ($_m) { $_scheme = $_m.Matches[0].Groups[1].Value }
    }
    if (!$_scheme -and (Test-Path $_wtCfg)) {
        $_scheme = (Get-Content $_wtCfg -Raw | ConvertFrom-Json).profiles.defaults.colorScheme
    }
    if ($_scheme) {
        $env:COLORFGBG = if ($_scheme -match 'light') { '0;15' } else { '15;0' }
    }
}
```

Verified with an isolated test script against the live `.wezterm.lua` — `GruvboxLight` correctly resolves to `COLORFGBG=0;15` before OMP ever gets a chance to race its own detection.

## The unglamorous part

Committing the fix surfaced a second, unrelated problem: the file containing `Set-WeztermColorScheme` wasn't tracked in the dotfiles repo at all. It existed only as a plain file in `Documents`, born from a refactor that split shared aliases out of the main profile — a split that never got committed. On top of that, the profile file itself carried 89 lines of accumulated, working-but-uncommitted changes from earlier sessions. And a pre-commit secret scanner blocked the push over a hardcoded personal path in an unrelated `Bash` helper function, twice, because the same hardcoded path existed in two copies of that function in two different files.

None of that was the bug. All of it had to get cleared before the bug fix could actually land somewhere durable. This is the part that doesn't show up in "here's the one-line fix" writeups: half the actual time went to making sure the fix wouldn't just evaporate the next time a file got regenerated from a stale, unversioned copy.

## Takeaways

- **A working detection mechanism for terminal A is not a working detection mechanism for terminal B.** If a fix reads a specific application's config file, it silently stops applying the moment the environment moves to a different application — with no error, just quiet non-coverage.
- **Env vars set by a switcher function only help processes started *after* the switch.** If you want a live-editing theme switcher to actually inform other tools, it has to touch the same signal those tools read, not just the thing it visually changes.
- **"We fixed this before" is a hypothesis, not a fact.** Check what changed in the environment since that fix landed before assuming it still covers the current case.
- **Untracked "helper" files accumulate real, working logic with zero backup.** If a refactor moves code into a new file, that file needs to enter version control in the same breath — not after the next person trips over it being unrecoverable.

## Source

- Fix committed to `ankitg12/dotfiles` (`Microsoft.PowerShell_profile.ps1`, `ps-shared-aliases.ps1`)
- OMP theme resolution: `packages/coding-agent/src/modes/theme/theme.ts` in `can1357/oh-my-pi`
- Related prior post: {% post_url 2026-07-03-hora-aware-ai-agent-omp %}
