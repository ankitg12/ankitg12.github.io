---
layout: post
title: "The Console Flash and the Watchdog That Lied About It"
date: 2026-08-14 04:30:00 +0530
categories: windows powershell debugging automation
---

A PowerShell scheduled task flashed a console window on my desktop every five minutes. Chasing it
cost two hours, produced three retractions, and ended somewhere I did not expect: a one-line bug
that had been making a Wi-Fi watchdog report success for work it never did.

The window was the symptom. The interesting part is everything I got wrong on the way.

## The obvious fix that isn't

Seven scheduled tasks, all launching PowerShell. Six were already written the way every Stack
Overflow answer tells you to write them:

```xml
<Command>powershell.exe</Command>
<Arguments>-WindowStyle Hidden -NoProfile -File C:\tools\watchdog.ps1</Arguments>
```

One was not — no `-WindowStyle Hidden` at all. Found it, fixed it, declared victory.

Then the window appeared again. Twice.

`-WindowStyle Hidden` **cannot** suppress the flash, and this is documented behaviour, not a
mystery: Windows allocates the console for `powershell.exe` *before* PowerShell parses its own
command line. By the time the `-WindowStyle` argument is read, the window already exists. The
switch hides it retroactively. You see the flash.

[PowerShell/PowerShell#3028](https://github.com/PowerShell/PowerShell/issues/3028) has been open
since 2017 for exactly this.

## The fix that ships with Windows

The tools people reach for here are [Hidden Start](https://www.ntwind.com/software/hstart.html)
(2007) and NirCmd's `exec hide` (2003) — both genuinely good, both pre-dating the current era of
generated utilities. I skipped both, because the capability is already in the box.

`wscript.exe` is a Windows Script Host that has no console. A process it launches with
`WshShell.Run(cmd, 0, True)` never gets one — not hidden, *never created*:

```vbscript
' run-hidden.vbs
Set sh = CreateObject("WScript.Shell")
WScript.Quit sh.Run(WScript.Arguments(0), 0, True)
```

```xml
<Command>wscript.exe</Command>
<Arguments>//B //Nologo C:\tools\run-hidden.vbs "powershell.exe -nop -file C:\tools\watchdog.ps1"</Arguments>
```

Two details in that four-line script are load-bearing, and I got both wrong first.

**`True`, not `False`.** [`WshShell.Run`](https://ss64.com/vb/run.html)'s third parameter is
`bWaitOnReturn`. With `False`, `wscript` exits immediately and Task Scheduler records the task as
completed while your job is still running. That silently destroys three things: `Last Result`
stops describing the job, `MultipleInstancesPolicy` (`IgnoreNew`) stops preventing overlapping
runs because there is no running instance to detect, and any watchdog logic that checks
`$task.State -eq 'Running'` goes blind. `WScript.Quit` then propagates the child's exit code so
`Last Result` stays truthful.

**One opaque argument, not a rebuilt one.** My first version looped over `WScript.Arguments` and
rejoined them, re-quoting anything containing a space. That is lossy: `WScript.Arguments` has
already discarded the original quoting, so embedded quotes, empty arguments, and shell
metacharacters cannot be recovered. Pass the entire command line as a *single* argument, quoted
once at the task-XML boundary, and read it as `Arguments(0)`.

## Verifying "no window" without trusting your eyes

Absence of complaints is not evidence of absence. The check is cheap — enumerate visible
top-level windows across a run and look for console classes:

```python
import ctypes, ctypes.wintypes as wt
u = ctypes.windll.user32

def visible_consoles():
    found = []
    @ctypes.WINFUNCTYPE(ctypes.c_bool, wt.HWND, wt.LPARAM)
    def cb(h, l):
        if u.IsWindowVisible(h):
            b = ctypes.create_unicode_buffer(256)
            u.GetClassNameW(h, b, 256)
            if b.value in ('ConsoleWindowClass', 'CASCADIA_HOSTING_WINDOW_CLASS'):
                found.append(b.value)
        return True
    u.EnumWindows(cb, 0)
    return found

base = visible_consoles()
# ... trigger the task, then poll at 20Hz for 25s ...
```

`ConsoleWindowClass` is the classic console host; `CASCADIA_HOSTING_WINDOW_CLASS` is Windows
Terminal. Baseline first, then diff — otherwise your own terminal shows up as a false positive.
Result across a full task run: no new console windows. That is a measurement, not a vibe.

## The part I did not go looking for

Same night, the machine's Wi-Fi dropped. A separate scheduled task exists for this: Windows'
Network Connectivity Status Indicator (NCSI) raises an event when the gateway stops answering ARP,
and the task disconnects and reconnects the adapter.

Its log said it worked:

```
03:06:23  Recovery starting
03:06:27  ERROR  Connect command failed: network not available
03:06:36  RECOVERED  gateway='192.168.29.1'
```

Read that again. The reconnect **failed**, and nine seconds later the same script logged
`RECOVERED` and exited 0. There is no retry loop in the code. `Invoke-WlanReconnect` returns
`$false` on that error and the caller is supposed to exit non-zero.

I assumed a second task instance had run. The Task Scheduler operational log settled it: exactly
one instance, `{04103172-…}`, started 03:05:44, event 201 completed 03:06:36 with return code 0.
Event 322 showed a second NCSI trigger arriving at 03:06:06 and being *refused* by `IgnoreNew`. So
one instance logged both the failure and the success.

The cause was on line 82 of the script's logger:

```powershell
function Write-WatchdogLog {
    param($Message, $Level = 'INFO')
    $Line = "$(Get-Date -Format o) [$Level] $Message"
    Add-Content -Path $LogPath -Value $Line
    Write-Output $Line          # <-- this
}
```

In PowerShell, [everything a function writes to the success stream is part of its return
value](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_return).
`Write-Output` in a logger means every caller that logs before returning gets the log line
prepended to its result:

```powershell
function Reconnect {
    Write-WatchdogLog 'Connect command failed' 'ERROR'
    return $false
}

if (Reconnect) { 'took this branch' }   # <-- it does
```

`Reconnect` does not return `$false`. It returns `@('2026-08-14T03:06:27 [ERROR] Connect command
failed', $false)` — a two-element array, which PowerShell evaluates as **`$true`**. The caller
concluded the reconnect had succeeded, slept 8 seconds, re-probed the gateway, found it alive, and
logged `RECOVERED`.

The gateway *was* alive. Windows WLAN AutoConfig had re-associated on its own at 03:06:31, which
the WLAN operational log records as event 8001. The script had disconnected the adapter, failed to
reconnect it, and then taken credit for the operating system's recovery. When AutoConfig lost that
race — which it does — the link simply stayed down and I reconnected by hand, without ever
suspecting the thing whose job was to prevent that.

The fix is one word:

```powershell
Write-Verbose $Line    # not Write-Output
```

And the test is the branch that actually broke, not the happy path:

```powershell
function T { Write-WatchdogLog 'simulated failure' 'ERROR'; return $false }
'elements=' + @(T).Count      # 1, was 2
'type='     + (T).GetType().Name   # Boolean, was Object[]
if (T) { 'BROKEN' } else { 'OK' }  # OK
```

## What generalises

**`Last Result: 0` means the script exited zero. Nothing more.** It is not evidence the script did
its job, and for a script that can talk itself into `RECOVERED`, it is not even evidence the job
was needed.

**A logger that writes to the success stream will corrupt every boolean-returning function that
calls it.** This is silent, it is not a syntax error, and it fails in the direction of false
confidence. Use `Write-Verbose`, `Write-Information`, or `Write-Host` — anything but
`Write-Output`. [PowerShell's output streams](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_output_streams)
are worth ten minutes of reading if you write scripts that return values.

**Hiding a window is not the same as fixing lifecycle.** Every one of my first three fixes made
the symptom better and the supervision worse. The shim that suppressed the flash also would have
made Task Scheduler lie about whether the job was running. The retry loop I added to the Wi-Fi
script still has not been executed once, so I have not claimed it works and the task stays
disabled until it is tested against a real disconnect.

**Correlate against timestamps, not narrative.** Each time I asserted a cause, the fix was to pull
the actual event log and match it against observed times. Task Scheduler's operational channel
(`Microsoft-Windows-TaskScheduler/Operational`, events 129 launch / 201 completion / 322
refused-by-policy) and `Microsoft-Windows-WLAN-AutoConfig/Operational` (8001 connected, 8003
disconnected, 11001 association) between them explained every event of the night, and contradicted
me three times.

## Source

- [PowerShell/PowerShell#3028 — `-WindowStyle Hidden` still flashes a console](https://github.com/PowerShell/PowerShell/issues/3028)
- [`WshShell.Run` reference — `intWindowStyle` and `bWaitOnReturn`](https://ss64.com/vb/run.html)
- [about_Return — what a PowerShell function actually returns](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_return)
- [about_Output_Streams](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_output_streams)
- [`Write-Verbose`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/write-verbose)
- [`EnumWindows` (Win32)](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-enumwindows)
- [`schtasks` reference](https://ss64.com/nt/schtasks.html)
- [Task Scheduler API](https://learn.microsoft.com/en-us/windows/win32/taskschd/task-scheduler-start-page)
