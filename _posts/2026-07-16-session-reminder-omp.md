---
layout: post
title: "session-reminder-omp: when the API you want doesn't exist, build the API you can"
date: 2026-07-16
categories: ai agents productivity
series: "AI coding agent productivity"
---

I run `/shake` by hand every so often — a mechanical, non-LLM operation in my
coding agent that trims images and stale tool output from the live context
without touching the model's understanding of the conversation. My agent's
`compaction.strategy: shake` already auto-fires it once a session crosses a
token threshold, but that's size-triggered. What I actually wanted was
time-triggered: a nudge every 10 minutes, independent of how big the context
has gotten.

The obvious move — write an extension that calls `session.shake()` on a
timer — doesn't work, and figuring out *why* was more interesting than
building the workaround.

## Chasing an API that isn't there

Extensions in my harness (OMP, a Claude Agent SDK-based coding tool) get an
`ExtensionContext` with a `compact(options)` method. Its `mode` union is
`"soft" | "remote" | "snapcompact"` — no `"shake"`. The real `/shake` command
handler calls `runtime.session.shake(mode)` directly, where `runtime.session`
is the full `AgentSession` object. Extensions don't get that — they get a
read-only `sessionManager`.

The next idea: just send the literal text `"/shake"` as a user message and
let the command parser pick it up.

```ts
ctx.sendUserMessage("/shake"); // doesn't do what you think
```

This also doesn't work, and not by accident. `sendUserMessage` calls
`session.prompt(text, { expandPromptTemplates: false })` — command parsing
is deliberately skipped for extension-originated messages. Sending `"/shake"`
this way delivers the literal four characters to the model as text, not as a
command. That's almost certainly intentional: an extension silently
triggering arbitrary slash commands (including ones with side effects) on
your behalf would be a much scarier attack surface than "can't auto-shake."

Once both paths dead-ended, I filed the gap upstream
([`can1357/oh-my-pi#5661`](https://github.com/can1357/oh-my-pi/issues/5661))
rather than reaching for something worse — monkeypatching internals, or
forking the session controller. If the extension API doesn't expose
something, the fix is to ask for the API, not to route around the boundary
that's (probably) there on purpose.

## Building the part that *is* buildable

What extensions **can** do: register lifecycle hooks (`session_start`,
`session_shutdown`), run a plain `setInterval`, and call `ctx.notify()` /
`ctx.setStatus()`. That's enough to build a reminder — just not one that
executes anything, only one that tells the human to.

First pass was hardcoded to `/shake`, 10 minutes, one message. That's a
smell: a single-purpose extension for a pattern ("periodic human-facing
nudge") that's obviously going to want more instances the moment someone
thinks about it for five more seconds. So the second pass made the whole
thing config-driven:

```json
{
	"reminders": [
		{ "name": "shake", "message": "…run /shake to reclaim tokens.", "intervalMs": 600000, "icon": "⏰" },
		{ "name": "stretch", "message": "Stand up and stretch.", "intervalMs": 3600000, "icon": "🧘" },
		{ "name": "eyes", "message": "20-20-20: look 20ft away for 20s.", "intervalMs": 1200000, "icon": "👀" },
		{ "name": "hydrate", "message": "Drink some water.", "intervalMs": 2700000, "icon": "💧" },
		{ "name": "posture", "message": "Shoulders back, feet flat, screen at eye level.", "intervalMs": 1800000, "icon": "🧍" }
	]
}
```

Each entry arms its own `setInterval` on `session_start` and clears on
`session_shutdown`. Malformed entries (missing `intervalMs`, non-positive
values, duplicate names) are skipped individually with a logged reason
rather than crashing the whole extension — one bad reminder shouldn't take
down the other four.

```ts
function arm(entry: ReminderConfig, ctx: ExtensionContext): Timer {
	return setInterval(() => {
		ctx.notify(entry.message, "info");
		ctx.setStatus(entry.name, `${entry.icon ?? "⏰"} ${entry.name}`);
		setTimeout(() => ctx.setStatus(entry.name, undefined), entry.flashMs ?? 15_000);
	}, entry.intervalMs);
}
```

## The 20-20-20 rule isn't as settled as I assumed

While writing the wellness reminders, I did the thing I'd tell anyone else
to do before shipping a "best practice" as a default: checked whether it's
actually backed by anything. The 20-20-20 rule (every 20 minutes, look at
something 20 feet away for 20 seconds) turns out to have mixed evidence in
recent ophthalmology literature — it measurably reduces *accommodative*
fatigue (the eye's focusing muscles), but does little for the dry-eye and
reduced-blink-rate symptoms that dominate actual digital eye strain
complaints (PubMed [35963776](https://pubmed.ncbi.nlm.nih.gov/35963776/),
[36473088](https://pubmed.ncbi.nlm.nih.gov/36473088/)). Separately, a 2025
comparison of break-taking techniques found structured Pomodoro/Flowtime
cadences outperform unstructured self-regulated breaks for sustained focus.

None of that changes the reminder I shipped — 20-20-20 is still a
reasonable default, cheap to follow, and better than nothing — but it's a
good reminder (pun intended) that "everyone knows you should do X every 20
minutes" is worth five minutes of verification before it becomes a default
in someone's tooling.

## Source

- [`session-reminder-omp`](https://github.com/ankitg12/session-reminder-omp) — the extension
- [`can1357/oh-my-pi#5661`](https://github.com/can1357/oh-my-pi/issues/5661) — upstream feature request to expose `shake()` to extensions
