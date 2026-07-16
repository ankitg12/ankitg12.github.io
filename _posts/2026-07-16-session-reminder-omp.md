---
layout: post
title: "session-reminder-omp: periodic reminders inside a coding-agent session"
date: 2026-07-16
categories: ai agents productivity tools
series: "AI coding agent productivity"
---

I was looking for an extension to periodically remind me of things during a coding-agent session — not calendar-style reminders, but ones scoped to a live session's wall clock. Concretely: I run `/shake` by hand every so often, a mechanical operation in my coding agent (OMP) that trims stale tool output and images from the live context without touching the model's understanding of the conversation. My agent's `compaction.strategy: shake` config already auto-fires that once a session crosses a token threshold — but that's size-triggered. What I wanted was time-triggered: a nudge every 10 minutes, independent of how big the context has gotten.

I couldn't find an existing extension that did this, so I built one: [`session-reminder-omp`](https://github.com/ankitg12/session-reminder-omp).

## What it does

It's a small OMP extension that fires arbitrary wall-clock-timed nudges — a notification plus a brief status-bar flash — on whatever cadence you configure, for as long as a session is open. Each reminder is independent: its own name, message, interval, icon, and flash duration. Nothing about it is specific to `/shake`; that's just the first thing I needed a nudge for.

Reminders live in a plain JSON config file, `~/.omp/agent/session-reminder.json`:

```json
{
	"reminders": [
		{ "name": "shake", "message": "Context has been growing for 10min — consider running /shake to reclaim tokens.", "intervalMs": 600000, "icon": "⏰" },
		{ "name": "stretch", "message": "You've been at this an hour — stand up and stretch.", "intervalMs": 3600000, "icon": "🧘" },
		{ "name": "eyes", "message": "20-20-20: look at something 20 feet away for 20 seconds.", "intervalMs": 1200000, "icon": "👀" },
		{ "name": "hydrate", "message": "Drink some water.", "intervalMs": 2700000, "icon": "💧" },
		{ "name": "posture", "message": "Posture check — shoulders back, feet flat, screen at eye level.", "intervalMs": 1800000, "icon": "🧍" }
	]
}
```

Once I had the mechanism, the obvious move was to stop thinking of it as a shake-nudge tool and use it for what it actually is: a generic "tell me things periodically while I'm sitting here" utility. So alongside the shake reminder, my own config now nudges me to stretch, blink/rest my eyes, drink water, and check my posture — all things I already know I should do and reliably forget to, because nothing in my environment was timed to a session's actual duration.

Each entry arms its own timer on session start and clears it on session end:

```typescript
function arm(entry: ReminderConfig, ctx: ExtensionContext): Timer {
	return setInterval(() => {
		try {
			ctx.ui.notify(entry.message, "info");
			ctx.ui.setStatus(entry.name, `${entry.icon ?? "⏰"} ${entry.name}`);
			setTimeout(() => ctx.ui.setStatus(entry.name, undefined), entry.flashMs ?? 15_000);
		} catch (e) {
			// one bad reminder logs and skips — it doesn't take the others down
		}
	}, entry.intervalMs);
}
```

Malformed config entries — missing `intervalMs`, a non-positive interval, a missing `message` — are skipped individually with a logged reason rather than failing the whole extension. One bad reminder shouldn't cost you the other four.

## The one gap worth naming

The extension nudges; it doesn't act. There's currently no supported way for an OMP extension to actually *invoke* the underlying `/shake` operation programmatically — only the built-in command controller can call that directly, so even the shake reminder above is a "go run this yourself" message, not an auto-triggered one. I filed that as a feature request upstream ([`can1357/oh-my-pi#5661`](https://github.com/can1357/oh-my-pi/issues/5661)) rather than reach for something uglier like monkeypatching the session internals. If an extension API doesn't expose something, asking for the API beats routing around the boundary.

## A bug that taught me something about extension crash isolation

Worth a specific callout since it surprised me: my first version called `ctx.notify(...)` and `ctx.setStatus(...)` directly — except those methods don't exist at that path; they live under `ctx.ui.notify`/`ctx.ui.setStatus`. Every timer tick threw a `TypeError`, and because that throw happened inside a `setInterval` callback I'd scheduled myself — not inside the extension's `session_start` handler invocation, which OMP does wrap in try/catch — it escaped all containment and crashed the entire session, not just the misbehaving extension.

That's a sharper edge than it sounds: any extension that schedules its own timers or async work (which the SDK explicitly supports, and which this exact use case requires) can currently take the whole harness down on a runtime error in that code. I fixed my own extension (correct API path, defensive try/catch around every fire callback) and filed the underlying gap separately ([`can1357/oh-my-pi#5664`](https://github.com/can1357/oh-my-pi/issues/5664)) since it's a real isolation hole, not something specific to my mistake.

## Whether the reminders are even good advice

Before shipping the wellness set as defaults, I checked whether "everyone knows" habits like the 20-20-20 rule actually hold up. Turns out it's more mixed than folklore suggests: recent ophthalmology literature shows it measurably helps *accommodative* eye fatigue (the focusing muscles) but does little for the dry-eye/reduced-blink-rate symptoms that dominate actual digital eye strain complaints (PubMed [35963776](https://pubmed.ncbi.nlm.nih.gov/35963776/), [36473088](https://pubmed.ncbi.nlm.nih.gov/36473088/)). Separately, a 2025 comparison of break-taking techniques found structured Pomodoro/Flowtime cadences outperform unstructured self-regulated breaks for sustained focus. None of that changes what I shipped — 20-20-20 is still cheap and better than nothing — but "everyone knows you should do X every 20 minutes" was worth five minutes of verification before it became a default in my own tooling.

## Source

- [`session-reminder-omp`](https://github.com/ankitg12/session-reminder-omp) — the extension
- [`can1357/oh-my-pi#5661`](https://github.com/can1357/oh-my-pi/issues/5661) — feature request: expose `shake()` to extensions
- [`can1357/oh-my-pi#5664`](https://github.com/can1357/oh-my-pi/issues/5664) — bug: extension-scheduled callbacks aren't crash-isolated
</content>
