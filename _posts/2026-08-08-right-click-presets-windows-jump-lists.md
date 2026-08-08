---
layout: post
title: "One Shortcut, Five Launch Modes: Right-Click Presets on Windows"
date: 2026-08-08 15:30:00 +0530
categories: windows tools productivity ai agents
series: "AI coding agent productivity"
---

A Windows shortcut holds exactly one command line. My agent launcher needed five.

The shortcut in question starts a coding agent in a clean sandbox: WezTerm opens at a scratch workspace and runs a PowerShell launcher that invokes [Oh My Pi (OMP)](https://github.com/can1357/oh-my-pi) with extensions, skills, and rules disabled. That bare mode is the default for a reason. But perhaps one launch in four, I want extensions on, or a different model, or higher thinking effort — and editing the `.lnk` arguments each time is not a workflow.

The obvious ask: right-click the Start menu entry and pick a mode. Here is what that actually takes on Windows, and the two bugs it surfaced in code that had been "working" for weeks.

## Right-click menus are not a property of the shortcut

The first instinct is wrong. You cannot attach options to a `.lnk` file. What you see when you right-click an app on the taskbar is a **Jump List**, and Jump Lists are keyed to an **AppUserModelID** (AUMID) — an identity string — not to a shortcut file.

The `Tasks` section is the part I wanted. Microsoft's framing is precise: [a destination is a noun, a task is a verb](https://learn.microsoft.com/en-us/windows/win32/shell/taskbar-extensions). Tasks are application-defined actions, each one a shell link with its own arguments.

This matters because my shortcut's target is `wezterm-gui.exe`. Register tasks naively and they attach to WezTerm's identity, polluting every WezTerm window on the system. The fix is documented: give the shortcut its **own** explicit AUMID via the `System.AppUserModel.ID` property, then register the task list under that same ID.

```
Agent Zero.lnk  ──(AUMID: AgentZero.Launcher)──▶  jump list
       │                                              │
       └─ target: wezterm-gui.exe                     ├─ With extensions
                                                      ├─ Model: opus
                                                      ├─ Extensions + opus
                                                      ├─ Deep thinking (xhigh)
                                                      └─ Choose at startup...
```

## You do not need a compiled helper

Every sample you will find implements this in C++ or C#, inside the application, at startup — the app owns its AUMID, registers its own list, and an installer stamps the ID onto the shortcut. That is the standard pattern, and it is why apps like Windows Terminal can offer per-profile entries.

But read the contract closely. `ICustomDestinationList` registers a list for whatever AUMID the **calling process** has adopted. Nothing requires that process to be the app. Any process that calls [`SetCurrentProcessExplicitAppUserModelID`](https://learn.microsoft.com/en-us/windows/win32/api/shobjidl_core/nf-shobjidl_core-setcurrentprocessexplicitappusermodelid) first can register the list — including PowerShell with `Add-Type` COM interop.

That collapses the whole thing into one script with no build step and no binary to maintain. The [call sequence](https://learn.microsoft.com/en-us/windows/win32/api/shobjidl_core/nn-shobjidl_core-icustomdestinationlist) is `BeginList` → `AddUserTasks` → `CommitList`:

```csharp
public static void Install(string aumid, string target, string[] titles, string[] argv,
                           string icon, string cwd) {
    SetCurrentProcessExplicitAppUserModelID(aumid);

    var list = (ICustomDestinationList)new DestinationList();
    list.SetAppID(aumid);

    uint slots;
    var iidArray = typeof(IObjectArray).GUID;
    object removed;
    Marshal.ThrowExceptionForHR(list.BeginList(out slots, ref iidArray, out removed));

    var tasks = (IObjectCollection)new EnumerableObjectCollection();
    for (int i = 0; i < titles.Length; i++)
        tasks.AddObject(MakeTask(target, argv[i], titles[i], icon, cwd));

    Marshal.ThrowExceptionForHR(list.AddUserTasks((IObjectArray)tasks));
    list.CommitList();
}
```

Two constraints the docs state and you will otherwise discover the hard way: every task link [must declare arguments](https://learn.microsoft.com/en-us/windows/win32/api/shobjidl_core/nf-shobjidl_core-icustomdestinationlist-addusertasks) — an argument-less link is rejected — and task entries cannot be pinned or removed by the user. Task titles are not `SetDescription`; they come from `PKEY_Title` on the link's `IPropertyStore`.

The honest cost of registering externally: the AUMID is not backed by a running process. WezTerm sets its own AUMID when it launches, so the new window does not group under the pinned icon. You get the menu; you do not get the grouping. A shim executable would fix that, and it buys nothing else.

## Where the menu actually appears

This is the part worth knowing before you build anything, because the answer changed recently.

| Location | Jump List shown? |
|---|---|
| Taskbar (pinned or running) | Yes — always has |
| Start menu, **pinned** tile | Yes, on Win11 22631.4534 / 26100+ |
| Start menu, "All apps" list | No |

Windows 11's Start menu was rewritten without Jump Lists and only regained them for pinned entries. If you build this and right-click your app in "All apps," you will see nothing and conclude the registration failed. It did not.

## Trap 1: the sandbox ate the shell's known folders

First run failed with `0x80070003` — `ERROR_PATH_NOT_FOUND` — from a COM call that touches no path I supplied.

Jump Lists are written to `%APPDATA%\Microsoft\Windows\Recent\CustomDestinations`. The shell resolves that through the known-folder APIs, which derive from `USERPROFILE`. I was running the installer *from inside the sandboxed agent session*, where `USERPROFILE` is redirected to the scratch workspace:

```
env:APPDATA      = C:\Users\<you>\AppData\Roaming      # correct
env:USERPROFILE  = C:\agent-zero                       # redirected
GetFolderPath    = C:\agent-zero\AppData\Roaming       # derived from USERPROFILE
registry AppData = C:\Users\<you>\AppData\Roaming      # ground truth
```

`APPDATA` was fine; `USERPROFILE` was not, and the known-folder path follows the latter. The registration was trying to write into a profile that does not exist — and even if the directory had existed, Explorer would never have read it.

The fix is to treat the registry as ground truth and refuse to trust the environment:

```powershell
$realAppData = (Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders').AppData
$realProfile = Split-Path (Split-Path $realAppData -Parent) -Parent
if ($env:USERPROFILE -ne $realProfile) {
    Write-Host "Restoring USERPROFILE: $env:USERPROFILE -> $realProfile"
    $env:USERPROFILE = $realProfile
}
```

Generalizable lesson: any tool that writes to a per-user shell location is unsafe to run inside a home-redirected sandbox. It will not error usefully. It will write somewhere plausible and silently wrong.

## Trap 2: `--model opus` was overwriting my workspace path

This one had nothing to do with Jump Lists, and it was the more valuable find.

To test the presets I instrumented the launcher to print the exact argument vector it hands to OMP. Passing flags through produced this:

```
[passthrough]  .\launch-omp.ps1 --model gpt-5.2 -c
  ARGV: --no-extensions --no-skills --no-rules --tools read,bash,... -c
```

`--model gpt-5.2` vanished. `-c` survived. Dumping the binder explains why:

```
bound : Workspace=--model, StateDir=gpt-5.2
args  : -c | --advisor
```

`--model` is not valid PowerShell parameter syntax. PowerShell does not pass it through and does not complain — it treats the token as **positional**, so `--model` landed in `-Workspace` and `gpt-5.2` in `-StateDir`. The launcher was then building its sandbox at a directory literally named `--model` and pointing agent state at `gpt-5.2`.

That script had been in daily use. Every double-dash flag ever passed to it was silently corrupting the sandbox paths and never reaching the agent.

The fix is to disable positional binding entirely and collect leftovers explicitly:

```powershell
[CmdletBinding(PositionalBinding = $false)]
param(
    [string]$Workspace = "C:\agent-zero",
    [string]$Preset,
    [switch]$Menu,
    [Parameter(ValueFromRemainingArguments = $true)][string[]]$Rest
)
```

Now the split is clean:

```
bound : Extensions=True, Workspace=D:\w
rest  : --thinking | high
```

If you write PowerShell wrappers around non-PowerShell CLIs, [`PositionalBinding = $false`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_advanced_parameters) plus `ValueFromRemainingArguments` should be your default posture, not a fix you apply after losing an afternoon.

## Jump Lists cannot take free text

The ceiling on this approach is hard: a task is a frozen command line. There is no text input in a Jump List, so "pick any OMP flag from the right-click menu" is not achievable at the shell level, ever. And presets combine multiplicatively — extensions × model × thinking effort is a table you do not want to enumerate as menu entries.

So the menu covers the combinations that earn their place, and one entry, `Choose at startup...`, hands control to a picker inside the terminal that also accepts raw flags:

```
  Agent Zero

    1) Agent Zero (bare)
    2) With extensions
    3) Model: opus
    4) Extensions + opus
    5) Deep thinking (xhigh)

  Enter = bare, number = preset, or type omp flags directly.
  >
```

Presets live in one `presets.json` that both the launcher and the installer read, so adding a mode is a JSON edit plus a re-run of the installer:

```json
{ "id": "ext-opus", "label": "Extensions + opus", "extensions": true,
  "omp": ["--model", "opus"], "task": true }
```

## Verify the registration, not the absence of errors

`CommitList` returning success does not prove the shell can see your list. Two checks close the loop — the on-disk artifact, and the property actually stamped on the shortcut:

```powershell
# 1. a jump list file was written for our AUMID hash
Get-ChildItem "$appdata\Microsoft\Windows\Recent\CustomDestinations" -Filter *.customDestinations-ms |
  Where-Object LastWriteTime -gt (Get-Date).AddMinutes(-5)
#   1ced32d74a95c7bc.customDestinations-ms  10322 bytes

# 2. the shortcut carries the AUMID, read back through the shell
$item.ExtendedProperty('System.AppUserModel.ID')
#   AgentZero.Launcher
```

For the launcher itself, the test that mattered was not "does it start" but "what argv does it construct." Instrumenting the exec line to print `$ompArgs` and driving all nine preset and passthrough combinations took a minute and caught a bug that months of successful launches had hidden. A wrapper that starts is not a wrapper that works.

## Source

- [Application User Model IDs](https://learn.microsoft.com/en-us/windows/win32/shell/appids) — why identity, not the shortcut file, owns the Jump List.
- [Taskbar Extensions](https://learn.microsoft.com/en-us/windows/win32/shell/taskbar-extensions) — the Tasks category and the noun/verb distinction.
- [`ICustomDestinationList`](https://learn.microsoft.com/en-us/windows/win32/api/shobjidl_core/nn-shobjidl_core-icustomdestinationlist) and [`AddUserTasks`](https://learn.microsoft.com/en-us/windows/win32/api/shobjidl_core/nf-shobjidl_core-icustomdestinationlist-addusertasks) — call order and the arguments-required constraint.
- [`SetCurrentProcessExplicitAppUserModelID`](https://learn.microsoft.com/en-us/windows/win32/api/shobjidl_core/nf-shobjidl_core-setcurrentprocessexplicitappusermodelid) — the call that lets a non-app process register a list.
- [about_Functions_Advanced_Parameters](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_advanced_parameters) — `PositionalBinding` and `ValueFromRemainingArguments`.
- [Oh My Pi](https://github.com/can1357/oh-my-pi) — the harness being launched.
- [WezTerm](https://wezterm.org/) — the terminal the shortcut opens.
