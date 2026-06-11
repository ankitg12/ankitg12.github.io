---
layout: post
title: "Three traps in one PowerShell profile edit: OneDrive redirect, wrong path, and the UTF-8 BOM cliff"
date: 2026-06-11
categories: windows powershell tools
---

I wanted to add WezTerm theme switching to my `tdark` / `tlight` PowerShell aliases — a ten-minute job. It took an hour. Three traps in sequence, each one invisible until you hit it.

## What I was building

WezTerm hot-reloads its config when `~/.wezterm.lua` changes on disk. So switching themes from the shell is a regex replace:

```powershell
function Set-WeztermColorScheme {
    param([Parameter(Mandatory=$true)][string]$Scheme)
    $cfg = "$HOME\.wezterm.lua"
    $content = Get-Content $cfg -Raw
    $updated = $content -replace "config\.color_scheme\s*=\s*'[^']*'", "config.color_scheme  = '$Scheme'"
    if ($updated -eq $content) { Write-Warning "No color_scheme line found in $cfg"; return }
    Set-Content $cfg $updated -Encoding utf8 -NoNewline
}

function tlight {
    Set-TerminalColorScheme -Scheme 'GruvboxLight'   # Windows Terminal
    Set-WeztermColorScheme  -Scheme 'GruvboxLight'   # WezTerm
}

function tdark {
    Set-TerminalColorScheme -Scheme 'Campbell'
    Set-WeztermColorScheme  -Scheme 'Batman'
}
```

The regex is safe: `color_scheme\s*=` won't match `color_schemes` because `color_schemes` has `s` before `=`, not whitespace.

```powershell
# Verify:
'config.color_scheme  = ''GruvboxLight''' -replace "config\.color_scheme\s*=\s*'[^']*'", "config.color_scheme  = 'Batman'"
# → config.color_scheme  = 'Batman'   ✓

'config.color_schemes = { }' -replace "config\.color_scheme\s*=\s*'[^']*'", 'SHOULD_NOT_MATCH'
# → config.color_schemes = { }        ✓
```

Straightforward. Then the traps started.

---

## Trap 1: Wrong profile file

My first instinct: edit `C:\Users\<you>\Documents\PowerShell\Microsoft.PowerShell_profile.ps1`. That's the PS7 profile and it already had `tlight`/`tdark` defined there. Updated them, added the WezTerm call, reloaded.

```
PS> tdark
✅ Windows Terminal settings updated. It should auto-reload; if not, press Ctrl+Shift+R.
```

Only one line. WezTerm didn't change. No error from `Set-WeztermColorScheme`. Ran `. $PROFILE`. Same result.

The output message was `✅ Windows Terminal settings updated...` — but my `Set-TerminalColorScheme` says `"OK: Windows Terminal color scheme set to..."`. Different text. Different function. I was editing the wrong file.

The actual profile:

```
PS> $PROFILE
C:\Users\<you>\OneDrive - <Company>\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1
```

**OneDrive redirects `My Documents` to `OneDrive - <Company>\Documents\`.** On a Windows machine with OneDrive for Business, `Documents` is not `C:\Users\<you>\Documents`. It's the OneDrive path. `$PROFILE` follows this redirect. Any tool or editor that resolves the "real" path without following the junction will edit the wrong file.

Rule: always confirm with `$PROFILE` before touching any profile file. Once per session. Not optional.

---

## Trap 2: `tdark` worked — but from where?

The OneDrive profile (`WindowsPowerShell\Microsoft.PowerShell_profile.ps1`) had its own standalone `tlight`/`tdark` — no dot-sourcing of a shared file, just inline definitions. The `tdark` in `C:\Users\<you>\Documents\PowerShell\...` (the PS7 profile) was irrelevant because Windows PowerShell 5 never loads PS7's profile directory.

PS5 (`powershell.exe`) → `WindowsPowerShell\Microsoft.PowerShell_profile.ps1`  
PS7 (`pwsh.exe`) → `PowerShell\Microsoft.PowerShell_profile.ps1`

Two separate profile directories. Two separate sets of function definitions. The PS7 edit did nothing for a PS5 shell.

Fixed the right file. Added `Set-WeztermColorScheme` inline, updated `tlight`/`tdark` to call it. Added a dot-source of `ps-shared-aliases.ps1` so both shells share the same baseline going forward.

---

## Trap 3: Emoji breaks PowerShell 5's parser

After reloading the profile — parse errors:

```
At ...\Microsoft.PowerShell_profile.ps1:172 char:23
+             Set-Item "env:$($Matches[1])" $Matches[2]
Unexpected token 'env:$($Matches[1])" $Matches[2] ...

At ...:198 char:51
+     $script = "$env:USERPROFILE\scoop\shims\b2p.py"
The string is missing the terminator: ".

At ...:44 char:18
+ function tscheme {
Missing closing '}' in statement block or type definition.
```

Lines 172 and 198 look valid. Line 44 (`tscheme`) has nothing obviously wrong. But all three errors appeared *after* my edit.

Parse-check with PS5's own parser:

```powershell
$f = 'C:\Users\<you>\...\Microsoft.PowerShell_profile.ps1'
$errors = $null
[System.Management.Automation.Language.Parser]::ParseFile($f, [ref]$null, [ref]$errors) | Out-Null
$errors | ForEach-Object { "Line $($_.Extent.StartLineNumber): $($_.Message)" }
```

Same three lines. PS7 parser: `OK: no parse errors`. The file is valid PS7, broken PS5.

The cause: **PowerShell 5 requires UTF-8 with BOM for script files containing non-ASCII characters.** Without BOM, PS5 assumes a legacy code page and misinterprets multi-byte sequences — emoji characters, em dashes, anything above U+007F. One bad byte breaks string parsing, and string errors cascade because the parser loses track of where strings begin and end. That's why `tscheme`'s `❌`/`✅` emoji at line 61/71 broke the `export` function at 172 and the `b2p` function at 198.

The file had a BOM before. My edit tool saved it without one. PS5 noticed; PS7 didn't.

**The fix:** resave via PS5, which writes UTF-8 with BOM by default:

```powershell
# Run in powershell.exe (PS5), not pwsh (PS7)
$f = 'C:\Users\<you>\...\Microsoft.PowerShell_profile.ps1'
$content = Get-Content $f -Raw -Encoding UTF8
Set-Content $f -Value $content -Encoding UTF8   # PS5: adds BOM. PS7: no BOM.
```

Parse-check again: `OK: no parse errors`.

This asymmetry is a real footgun:

| | `Set-Content -Encoding UTF8` |
|---|---|
| **PS5** (`powershell.exe`) | UTF-8 **with** BOM |
| **PS7** (`pwsh.exe`) | UTF-8 **without** BOM |

If your profile targets both, save it from PS5 or add a BOM explicitly.

---

## What actually ships

The OneDrive profile (`WindowsPowerShell\Microsoft.PowerShell_profile.ps1`):

1. Sources shared aliases first (coreutils wrappers, git helpers, etc.)
2. Defines `Set-WeztermColorScheme`, then `tlight`/`tdark` — the inline definitions override the shared ones if they differ
3. Saved as UTF-8 with BOM via PS5

The PS7 profile (`PowerShell\Microsoft.PowerShell_profile.ps1`):

1. Sources the same shared aliases
2. Used to define `tlight`/`tdark` inline — those are now removed; shared aliases carry them

Both terminals, both shells, one set of functions.

---

## Checklist for any profile edit

1. `$PROFILE` — confirm the actual path before opening any editor
2. Which shell is the target? PS5 (`powershell.exe`) and PS7 (`pwsh.exe`) load from different directories
3. Any non-ASCII in the file? If yes, save from PS5 or ensure BOM is present
4. Parse-check before reloading: `[System.Management.Automation.Language.Parser]::ParseFile($f, [ref]$null, [ref]$errors)`

The actual feature — two lines of `Set-Content` on a Lua config — took five minutes. The rest was reading the environment.
