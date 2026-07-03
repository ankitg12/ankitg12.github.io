---
layout: post
title: "Teaching an AI coding agent to know what time it is (the Jyotish way)"
date: 2026-07-03
categories: ai agents productivity
series: "AI coding agent productivity"
---

I already have an AFK extension that switches my coding agent into autonomous mode on command. The obvious next question: could the agent bias its *own* tone automatically, based on time of day — without me typing a command at all?

Jyotish (the Indian astronomical/astrological tradition) already has a centuries-old answer to "what should I be doing right now, time-wise": the **Hora** system. A civil day is divided into 24 planetary hours, each associated with an activity bias — Mercury for study and communication, Mars for high-effort action, Saturn for cleanup and maintenance. It's a clean, pre-built ontology for "what kind of work fits this moment" — so I built an extension that computes it for real and feeds it to the agent every turn as a prompt-level nudge, not a mode switch.

Three engineering lessons came out of it that are worth more than the feature itself: getting a hook's array-vs-string contract wrong fails silently, timezone bugs hide in `Date` methods you'd swear are timezone-agnostic, and "sticky until X" flags need every re-entry path audited, not just the happy one.

---

## What a Hora actually is

Not a fixed 60-minute slot. A civil day runs sunrise-to-sunrise, split into 12 daytime Horas (sunrise→sunset) and 12 nighttime Horas (sunset→next sunrise). Because day/night length varies by season and latitude, Hora length varies too — at my latitude (13°N) it runs 55–65 minutes year-round, but it's a real astronomical calculation, not a clock convention.

Each Hora is ruled by one of seven classical planets in a fixed repeating order:

```
Sun → Venus → Mercury → Moon → Saturn → Jupiter → Mars → (repeat)
```

The first Hora of each weekday is ruled by that weekday's own planetary lord — Sunday starts with Sun, Monday with Moon, and so on — which is *why* the classical weekday names line up with this sequence once you carry it across 24 Horas into the next sunrise.

## The build

Three small pieces:

**Sun-position math**, done locally with [suncalc](https://github.com/mourner/suncalc) — zero network calls, ~15s accuracy, validated against USNO. Given the civil-day anchor (walking back a day if `now` is before today's sunrise), dividing sunrise→sunset and sunset→next-sunrise into 12 equal slots each gives you the current Hora and how long until it ends.

```typescript
export function getCurrentHora(at: Date, coords: GeoCoords): HoraInfo {
	let civilDayAnchor = new Date(at);
	let times = SunCalc.getTimes(civilDayAnchor, coords.lat, coords.lng);
	if (at < times.sunrise) {
		civilDayAnchor = new Date(at.getTime() - 24 * 3600_000);
		times = SunCalc.getTimes(civilDayAnchor, coords.lat, coords.lng);
	}
	// ...divide sunrise→sunset and sunset→nextSunrise into 12 slots each
}
```

**Lesson 1 — a `Date` method you'd swear is timezone-agnostic isn't.** The weekday-ruler lookup needs the weekday **as observed at the configured location**, not the host machine's timezone. `Date.getDay()` always reads through local system time — wrong the moment your coordinates live in a different zone than the machine running the extension. Fixed with `Intl.DateTimeFormat` and an explicit `timeZone`:

```typescript
function weekdayInTimezone(at: Date, timezone: string): number {
	const dayName = new Intl.DateTimeFormat("en-US", { timeZone: timezone, weekday: "short" }).format(at);
	return ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"].indexOf(dayName);
}
```

`Date` objects don't carry a timezone — every method on them (`getDay`, `getHours`, ...) implicitly reads through whatever zone the *host process* is running in. The only zone-aware primitives in the standard library are `Intl.DateTimeFormat` and friends; anything else silently assumes "wherever this code happens to execute" is the zone that matters.

**A persona table**, one entry per planet with a tone hint and a short "gear" label (Mercury → "deep-work gears", Saturn → "cleanup gears"), distilled from traditional activity guidance.

**A scheduler** that arms a `setTimeout` directly to the next Hora boundary — never polls for the switch itself — plus a second timer for an optional pre-boundary reminder, and a 10-second interval that detects laptop sleep/wake (a >20s gap between ticks means the machine was suspended) and recomputes immediately rather than firing a stale timer.

## Injecting it: the part that actually matters

The status-bar glyph is decoration. The real mechanism is a hook that fires before every LLM call and appends a fragment to the system prompt:

```typescript
pi.on("before_agent_start", (event) => {
	if (!enabled || !currentPlanet) return undefined;
	const fragment = buildSystemPromptFragment(currentPlanet, { includeSuggestions: true });
	return { systemPrompt: [...event.systemPrompt, fragment] };
});
```

**Lesson 2 — an array-vs-string contract mismatch fails silently, not loudly.** `event.systemPrompt` here is a `string[]`, chained across extensions if more than one appends to it. Getting this wrong is easy: a community extension I found doing the same kind of system-prompt tone injection targeted an older version of the same coding-agent framework, where `systemPrompt` was a plain string and the correct pattern was `event.systemPrompt + myText`. Copy that code against a newer version and it doesn't crash — `[]string + string` in JS coerces the array to a comma-joined string first (`["a", "b"] + "c"` → `"a,bc"`), so instead of a clean append you get run-together, comma-mangled prompt text. No error, no type failure at the call site, just a subtly corrupted system prompt you'd only catch by actually reading what the model received.

**Always check the actual installed type definitions before trusting a tutorial or a sibling extension's code.** I typechecked against the real shipped `.d.ts` with `--strict --skipLibCheck` before writing a single hook, and it caught this exact class of drift, along with a stricter `Promise<void>` return-type requirement on command handlers that event handlers don't share — an asymmetry in the same SDK that only shows up under strict compilation against the real types, not from reading the extension-authoring guide.

Every injected fragment ends with the same line, non-negotiable:

> Always do exactly what the user explicitly asks regardless of Hora — this is flavor/bias, not a restriction.

This is deliberately weak enforcement. It's the same category as a Pomodoro timer or "maker's schedule vs manager's schedule" — a structured heuristic for time-boxing cognitive modes, not a claim that planetary hours have causal power over code quality. If you want something that actually blocks actions during a particular Hora, that's a different, stronger hook (`before_tool_call`-style gating) and a different risk profile — deliberately out of scope here.

## Proving it actually fires — not just typechecks

A hook that typechecks cleanly can still silently no-op at runtime. I wanted mechanical proof, not just a green compiler:

```typescript
function log(msg: string) { appendFileSync(LOG_FILE, `${new Date().toISOString()} ${msg}\n`); }
// ...
pi.on("before_agent_start", (event) => {
	if (!enabled || !currentPlanet) {
		debug(`before_agent_start: skipped (enabled=${enabled})`);
		return undefined;
	}
	debug(`before_agent_start: injecting ${currentPlanet} Hora fragment`);
	// ...
});
```

Then a real, throwaway session, driven by terminal automation rather than a mock:

```
before_agent_start: injecting Moon Hora fragment (553 chars)
```

And the model's actual response to an unrelated question echoed the injected persona's language ("reflective, low-drama completion work") — vocabulary that only exists in my persona table, not anywhere else in that session's context. That's the difference between "the code compiles" and "the mechanism works."

The trick for testing a "fires N minutes before deadline" scheduler without waiting out a real 20-minute boundary: compute the actual remaining time with the pure calculation function, then temporarily set the configured lead-time to just under that remaining value. Real remaining time of 17.3 minutes → set the reminder lead to 16 minutes → it fires in about 80 seconds, under real wall-clock conditions, in a real running session. Faster and more trustworthy than mocking the scheduler in isolation, because it exercises the actual hook plumbing, not just the arithmetic.

## The bug that only shows up on the unhappy path

**Lesson 3 — "sticky until X" flags need every re-entry path audited, not just the one you were thinking about.** The reminder needed to persist as a "prepare the agent" nudge across every turn until the actual switch — not just the one turn right after it fires, in case that turn is a two-word aside that doesn't touch the pending work. First attempt cleared the pending flag inside the scheduling function on every re-arm:

```typescript
function armTimer(ctx) {
	pendingReminderPrime = undefined; // WRONG — clears unconditionally
	// ...
}
```

Looks fine. Passes the happy-path test: reminder fires, prime shows up on the next turn, real boundary passes, new persona kicks in. The bug only surfaces when something *else* re-runs the same scheduling function mid-Hora — a sleep/wake recompute, or a manual "retry the config" command — which silently drops the prime before the real transition it was supposed to survive until.

The fix: capture the previous state before recomputing, clear only on an actual transition.

```typescript
function armTimer(ctx) {
	const previousPlanet = currentPlanet;
	const hora = getCurrentHora(new Date(), coords);
	currentPlanet = hora.planet;
	if (previousPlanet !== undefined && previousPlanet !== hora.planet) {
		pendingReminderPrime = undefined; // only clear on a REAL switch
	}
	// ...
}
```

General shape of the bug, useful beyond this one extension: for any "sticky until X" flag driven by a scheduler, audit *every* code path that can re-enter the scheduling function — not just the one you were thinking about when you wrote the flag.

## Where this sits, deliberately

What's built: real sunrise/sunset-based time computation, prompt-bias injection with mechanical proof it fires, a configurable pre-boundary reminder with a gear-specific message, sleep/wake resilience.

What's explicitly *not* built, on purpose: model or reasoning-effort switching per Hora, tool gating, or task-list filtering by Hora-matching tags. Those all change the risk profile — a bug in a prompt-bias fragment produces slightly-off tone; a bug in tool gating can block real work. Worth its own separate review if I build it at all.

## Source

- Local sunrise/sunset: [suncalc](https://github.com/mourner/suncalc) — check a library's full version history, not just the latest version number, before deciding it's "too new" to depend on. suncalc's `2.0.0` looked recent; the project goes back to `1.0.0` years earlier — the recent release was a precision rewrite, not evidence of an immature library.
- Hora reference: [drikpanchang.com/muhurat/hora](https://www.drikpanchang.com/muhurat/hora.html)
- Prior post in this series: {% post_url 2026-05-27-afk-toggle-for-ai-coding-agents %}
