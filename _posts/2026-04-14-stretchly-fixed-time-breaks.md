---
layout: post
title: "Fixed-time breaks with Stretchly on Windows"
date: 2026-04-14
categories: windows productivity
---

[Stretchly](https://hovancik.net/stretchly/) counts down from the moment it starts. If you launch it at 9:03, your breaks land at 9:13, 9:23, 9:33. Share a calendar with anyone and those times mean nothing. A break at 9:00, 9:10, 9:20 — on the clock — is a different thing.

This is [a known gap](https://github.com/hovancik/stretchly/issues/1638). The maintainer declined to implement it. None of the 22 active forks implement it either.

Here is how to add it on Windows without touching Stretchly's source.

---

## How it works

Windows Task Scheduler fires a task at every clock-aligned N-minute boundary. The task calls `stretchly mini` — Stretchly's own CLI — which triggers a break in the already-running app.

Two files:

**`stretchly-fixed-break.ps1`** — the task payload. Checks if Stretchly is running; calls `stretchly mini` if so, exits silently if not. Closing Stretchly is your pause mechanism.

**`setup-stretchly-fixed-breaks.ps1`** — registers the scheduled task. Run once; survives reboots.

---

## The trigger pattern

`New-ScheduledTaskTrigger` supports `-RepetitionInterval` only on its `-Once` parameter set, not `-Daily`. But a `Once` trigger with a fixed start date drifts — after its duration window expires, it stops firing.

The fix: create a `-Daily` trigger as the base (resets at midnight, fires every day), borrow the `Repetition` CimInstance from a `-Once` trigger, and assign it across before registering. Omitting `-RepetitionDuration` leaves the duration empty, which Task Scheduler treats as indefinite.

```powershell
$daily = New-ScheduledTaskTrigger -Daily -At 00:00
$rep   = New-ScheduledTaskTrigger -Once  -At 00:00 `
             -RepetitionInterval (New-TimeSpan -Minutes 10)
$daily.Repetition = $rep.Repetition
```

Task Scheduler shows this correctly in the UI: *"At 12:00 AM every day — after triggered, repeat every 10 minutes."*

`StartWhenAvailable:$false` ensures missed firings (machine asleep) are skipped rather than caught up, preserving clock alignment.

---

## Setup

```powershell
pwsh -File .\setup-stretchly-fixed-breaks.ps1
# or choose a different interval:
pwsh -File .\setup-stretchly-fixed-breaks.ps1 -IntervalMinutes 5
# valid values: 5, 10, 15, 20, 30
```

Verify it took:

```powershell
(Get-ScheduledTask -TaskName StretchlyFixedBreaks).Triggers[0].Repetition
Get-ScheduledTask -TaskName StretchlyFixedBreaks | Get-ScheduledTaskInfo | Select-Object NextRunTime
```

To remove:

```powershell
pwsh -File .\setup-stretchly-fixed-breaks.ps1 -Remove
```

---

## One thing to know about pause

Pausing breaks in Stretchly does not suppress these scheduled breaks. The `stretchly mini` CLI calls `skipToMicrobreak()` directly, without checking `breakPlanner.isPaused`. The tray UI hides the skip option when paused — the CLI does not have that guard (`app/main.js`, v1.18.1).

Closing Stretchly is the actual suppression mechanism. The task payload checks `Get-Process Stretchly` and exits silently if nothing is running, so the break is a no-op when the app is closed.

Enable *Launch at login* in Stretchly Preferences so it is always running when you are logged in. Closing it then acts as a manual pause for both its internal timer and the scheduled breaks.

---

## Disable Stretchly's own mini-break timer

Stretchly's internal timer runs independently. Without disabling it, you get two break sources. Go to Preferences → Mini Breaks and turn mini-breaks off. Leave long breaks on Stretchly's internal schedule if you want them on a different cadence.

---

Scripts are in this [gist](https://gist.github.com/ankitg12/d9d931e6e2f41506e456625a9ee0faa8).
