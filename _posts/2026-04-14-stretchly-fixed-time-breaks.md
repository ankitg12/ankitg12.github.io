---
layout: post
title: "Fixed-time breaks with Stretchly on Windows"
date: 2026-04-14
categories: windows productivity
---

[Stretchly](https://hovancik.net/stretchly/) counts down from the moment it starts. If you launch it at 9:03, your breaks land at 9:13, 9:23, 9:33. Share a calendar with anyone and those times mean nothing. A break at 9:00, 9:10, 9:20 — on the clock — is a different thing.

This is [a known gap](https://github.com/hovancik/stretchly/issues/1638). None of the 22 active forks implement it either.

Here is how to add it on Windows without touching Stretchly's source.

---

## How it works

Windows Task Scheduler fires a task at every clock-aligned N-minute boundary. The task calls `stretchly mini` — Stretchly's own CLI — which triggers a break in the already-running app.

**`stretchly-fixed-break.ps1`** — the task payload. Checks if Stretchly is running; calls `stretchly mini` if so, exits silently if not. Closing Stretchly is your pause mechanism.

---

## Setup

```powershell
schtasks /Create /SC MINUTE /MO 10 /TN "StretchlyFixedBreaks" `
  /TR "\"C:\Program Files\PowerShell\7\pwsh.exe\" -NonInteractive -WindowStyle Hidden -ExecutionPolicy Bypass -File \"%USERPROFILE%\tools\stretchly-fixed-break.ps1\"" `
  /ST 00:00 /F
```

`/SC MINUTE /MO 10 /ST 00:00` fires every 10 minutes starting from midnight — clock-aligned by construction. Survives reboots without any trigger window expiry.

Change the interval (replace `10` with `5`, `15`, `20`, or `30`):

```powershell
schtasks /Create /SC MINUTE /MO 5 /TN "StretchlyFixedBreaks" /TR "..." /ST 00:00 /F
```

Verify:

```powershell
schtasks /Query /TN "StretchlyFixedBreaks" /FO LIST
```

Remove:

```powershell
schtasks /Delete /TN "StretchlyFixedBreaks" /F
```

---

## One thing to know about pause

Pausing breaks in Stretchly does not suppress these scheduled breaks. The `stretchly mini` CLI calls `skipToMicrobreak()` directly, without checking `breakPlanner.isPaused`. The tray UI hides the skip option when paused — the CLI does not have that guard (`app/main.js`, v1.18.1).

Closing Stretchly is the actual suppression mechanism. The task payload checks `Get-Process Stretchly` and exits silently if nothing is running, so the break is a no-op when the app is closed.

Enable *Launch at login* in Stretchly Preferences so it is always running when you are logged in. Closing it then acts as a manual pause for both its internal timer and the scheduled breaks.

---

## Disable Stretchly's own mini-break timer

Stretchly's internal timer runs independently. Without disabling it, you get two break sources. Go to Preferences → Mini Breaks and turn mini-breaks off. Leave long breaks on Stretchly's internal schedule if you want them on a different cadence.

## Update — one clock, Stretchly as renderer only (August 2026)

Turning off only mini-breaks was not enough for my current setup: I wanted both mini and long breaks on clock boundaries, and leaving either native timer active eventually produced duplicate breaks.

Stretchly 1.22 makes the cleaner arrangement practical because its desktop installer exposes the CLI on `PATH`. Windows Task Scheduler now owns the entire schedule; Stretchly only renders the windows:

1. At login, start Stretchly and then run `stretchly pause -d indefinitely`. This cancels its native countdown while leaving the process available to receive CLI commands.
2. Every ten minutes, the scheduled task runs `stretchly mini`; at minute `:50`, it runs `stretchly long` instead.
3. A forced CLI break starts a new native countdown when it finishes, so the task waits for that break to end and runs `stretchly pause -d indefinitely` again. This re-pause is essential.
4. Set Task Scheduler's `MultipleInstancesPolicy` to `IgnoreNew`. A ten-minute long-break invocation remains alive until it re-pauses Stretchly; the next trigger must not overlap it.
5. Explicitly allow the task to start and continue on battery power. The default `schtasks /Create` battery policy is unsuitable for a laptop health reminder.

I use the regular desktop installer rather than the Microsoft Store package. The Store package has no `AppExecutionAlias`, and Windows does not permit these scripts to execute its packaged binary directly.

The task previously generated an AI summary of the last ten minutes and placed it in the break window. That was technically charming and psychologically wrong: a break is for looking inward, not interpreting another dashboard. The current script sends no title or activity text.

The maintained scripts are in [`ankitg-tools`](https://github.com/ankitg12/ankitg-tools):

- `setup-stretchly-fixed-breaks.ps1` creates the clock-aligned task and pins its overlap and battery policies in Task Scheduler XML.
- `stretchly-fixed-break.ps1` chooses mini versus long, triggers the plain break, then restores the indefinite pause.

Verification should use Stretchly's application log: look for the CLI command, break start, break finish, and the subsequent indefinite pause. Task Scheduler's `Last Result: 0` is insufficient—a PowerShell payload can log success even when an inner executable failed.

---

Script is in this [gist](https://gist.github.com/ankitg12/d9d931e6e2f41506e456625a9ee0faa8).
