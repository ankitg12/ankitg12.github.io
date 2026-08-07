---
layout: post
title: "Zen at Window 13: A Prompting Governor for Windows"
date: 2026-08-07
categories: windows tools productivity
---

Windows will arrange windows, switch between them, and throttle the processes behind them—but it will not tell you that opening window 13 should require closing one of the previous 12.

That missing policy matters. Overlapping windows hide accumulation: a browser popup here, another terminal there, three editor windows behind them. The cost becomes visible only when navigation is confusing or the machine is paging heavily.

Browser tab managers already understand this problem. A tab limit or least-recently-used eviction policy makes scarcity explicit. Desktop window managers generally stop at arrangement.

This post builds the missing layer: a small AutoHotkey v2 governor that prompts when a configurable window limit is crossed, identifies the stalest eligible window, and closes it only with explicit approval.

---

## The tools solve different layers

| Tool class | Controls | What it does not control |
|---|---|---|
| Tiling window manager | Placement and visibility | Number of windows |
| Process manager | CPU, memory, affinity, priority | Windows owned by a process |
| Virtual desktops | Context separation | Total resource use or accumulation |
| Prompting governor | Admission policy for new windows | Layout and process resources |

A process is not a window. One browser process tree may own many browser windows; one editor process may own several project windows. A process governor therefore cannot implement “opening one more window requires making room.”

A tiling manager such as [GlazeWM](https://github.com/glzr-io/glazewm) is still useful: it makes accumulation visible immediately because every new window consumes screen area. The prompting governor adds the missing admission policy.

---

## Policy before mechanism

The policy is deliberately small:

```text
limit = 12 eligible top-level windows

when the count increases beyond 12:
    find the least-recently-active eligible window
    ask whether to close it

Yes -> send the normal WM_CLOSE message
No  -> permit the new window and wait until another opens
```

Three constraints prevent this from becoming a desktop wrecking ball:

1. **Never kill a process.** Send `WM_CLOSE`, allowing the application to display its normal save confirmation.
2. **Never close without approval.** Desktop windows may hold unsaved documents, terminals, authentication dialogs, or long-running tasks. They are not as fungible as browser tabs.
3. **Allow pinning.** Some windows must never become eviction candidates even when they are old.

The limit is a calibration knob, not a universal constant. Twelve is enough to make scarcity real without turning every normal context switch into a negotiation.

---

## Count only user-facing windows

`WinGetList()` returns top-level windows, but not every returned handle represents something the user thinks of as a window. The filter excludes:

- invisible windows;
- empty-title windows;
- the desktop and taskbar;
- tool windows marked with `WS_EX_TOOLWINDOW`;
- the governor's own process.

```autohotkey
EligibleWindows() {
    global myPid
    result := []

    for hwnd in WinGetList() {
        try {
            if WinGetPID("ahk_id " hwnd) = myPid
                continue
            if !DllCall("IsWindowVisible", "ptr", hwnd)
                continue
            if Trim(WinGetTitle("ahk_id " hwnd)) = ""
                continue

            class := WinGetClass("ahk_id " hwnd)
            if class = "Progman" || class = "WorkerW"
                continue
            if class = "Shell_TrayWnd" || class = "Shell_SecondaryTrayWnd"
                continue

            if WinGetExStyle("ahk_id " hwnd) & 0x80 ; WS_EX_TOOLWINDOW
                continue

            result.Push(hwnd)
        }
    }
    return result
}
```

This is intentionally conservative. Browser-owned dialogs and application windows remain eligible if they look like ordinary top-level windows. A live soak against the actual application mix is still necessary before treating the rules as universal.

---

## Track recency rather than guessing importance

The governor samples the active window every 500 milliseconds and records its last-active tick count:

```autohotkey
active := WinExist("A")
if active && live.Has(active)
    lastSeen[active] := A_TickCount
```

For windows that existed before the governor started, Z-order supplies a reasonable bootstrap. `WinGetList()` is topmost-first, so windows nearer the bottom receive older synthetic timestamps:

```autohotkey
for index, hwnd in windows
    if !lastSeen.Has(hwnd)
        lastSeen[hwnd] := now - (index * 1000)
```

The candidate selector skips the active window and anything pinned:

```autohotkey
ChooseStalest(windows, active) {
    global lastSeen, pinned
    stale := 0
    oldest := 0x7FFFFFFFFFFFFFFF

    for hwnd in windows {
        if hwnd = active || pinned.Has(hwnd)
            continue
        seen := lastSeen.Get(hwnd, 0)
        if seen < oldest {
            oldest := seen
            stale := hwnd
        }
    }
    return stale
}
```

`Ctrl+Alt+P` toggles a pin on the focused window. Pinned windows still count toward the limit—they consume attention and resources—but are never offered for closure.

---

## Close normally, never terminate

The prompt names both the window and its owning executable:

```autohotkey
answer := MsgBox(
    "You now have " count " windows (Zen limit: " MAX_WINDOWS ").`n`n"
    "Close the stalest window?`n`n"
    WinGetTitle("ahk_id " stale) "`n["
    WinGetProcessName("ahk_id " stale) "]",
    "Window Governor — make room",
    "YesNo Icon? Default2 0x40000"
)

if answer = "Yes"
    PostMessage(0x0010, 0, 0, , "ahk_id " stale) ; WM_CLOSE
```

`Default2` makes **No** the default button. An accidental Enter therefore permits the new window rather than closing existing work.

`WM_CLOSE` asks the window to close through its normal application path. An editor can ask to save; a terminal can warn about running processes. `TerminateProcess` would bypass those protections and does not belong in this design.

---

## Prompt only on admission

A governor that repeats the same warning every 500 milliseconds is merely a nagging dashboard.

The prompt fires only when the eligible count increases:

```autohotkey
count := windows.Length
openedAnother := count > lastCount
lastCount := count

if count <= MAX_WINDOWS || !openedAnother || promptOpen
    return
```

If the user selects **No** at 13 windows, the governor remains quiet. Opening window 14 creates a new admission event and another decision point.

This places friction where the cost is introduced—not during an unrelated cleanup session hours later.

---

## Verification needs real windows

Two levels of checks caught different failures:

| Check | Covers |
|---|---|
| Deterministic self-test | LRU ranking, active-window exclusion, pinning |
| Disposable-window integration test | Window enumeration, limit crossing, prompt appearance, safe decline, cleanup |

The first integration fixture used modern Notepad and waited for the PID returned by `Run("notepad.exe")`. The window appeared, but the test failed: packaged Notepad uses a broker, so the launcher PID was not the window-owning PID.

That is a useful Windows testing lesson: **a PID returned by process launch is not always a stable window identity**. The corrected test launches a separately named AutoHotkey fixture window and finds it by its unique title.

Synthetic integration tests still do not cover every browser popup, brokered application, modal dialog, or multi-window editor. The safe deployment sequence is:

1. run deterministic and fixture tests;
2. run the governor interactively during normal work;
3. inspect every proposed candidate for false positives;
4. only then put it in the Startup folder.

---

## Start automatically without installing a service

AutoHotkey is already the runtime. A standard Startup shortcut is sufficient:

```powershell
$startup = [Environment]::GetFolderPath("Startup")
$path = Join-Path $startup "Window Governor.lnk"
$wsh = New-Object -ComObject WScript.Shell
$link = $wsh.CreateShortcut($path)
$link.TargetPath = "C:\Program Files\AutoHotkey\v2\AutoHotkey64.exe"
$link.Arguments = '"C:\Users\me\tools\window-governor.ahk"'
$link.WorkingDirectory = "C:\Users\me\tools"
$link.Save()
```

Deleting that one shortcut disables automatic startup. No service, scheduled task, registry policy, telemetry, or dashboard is required.

---

## Why polling is acceptable here

Windows exposes an event-driven alternative through `SetWinEventHook(EVENT_OBJECT_SHOW)`. It can wake the governor exactly when a window appears.

For a dozen windows, enumerating top-level handles twice per second is simpler and negligible. An event hook introduces callback lifetime, re-entrancy, and accessibility-event filtering. It is an upgrade only if measurement shows that polling consumes meaningful CPU or misses admission events.

Boring code that has passed its checks is preferable to a more elegant mechanism with a larger failure surface.

---

## Result

The useful abstraction is not another task manager. It is a small admission controller for attention:

```text
GlazeWM          -> makes every window visible
Window governor  -> makes every additional window deliberate
WM_CLOSE         -> preserves application safety
Pinning          -> protects intentional long-lived windows
```

The desktop remains flexible, but abundance stops being free and invisible. Window 13 is still allowed—it simply has to justify its place.

## Source

- [GlazeWM](https://github.com/glzr-io/glazewm)
- [AutoHotkey v2: WinGetList](https://www.autohotkey.com/docs/v2/lib/WinGetList.htm)
- [AutoHotkey v2: PostMessage](https://www.autohotkey.com/docs/v2/lib/PostMessage.htm)
- [AutoHotkey v2: MsgBox](https://www.autohotkey.com/docs/v2/lib/MsgBox.htm)
- [Microsoft: WM_CLOSE message](https://learn.microsoft.com/en-us/windows/win32/winmsg/wm-close)
- [Microsoft: SetWinEventHook](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwineventhook)
