---
layout: post
title: "The \$/1M-ctx-growth meter: measuring what a coding agent's context is actually costing you"
date: 2026-07-22
categories: ai agents productivity tools
series: "AI coding agent productivity"
---

A long-running AI coding agent session has a shape you don't notice until you've been bitten by it: cost grows roughly quadratically, because every tool round-trip resends the *entire* live context back to the model. A session that starts at 20k tokens and drifts to 150k isn't paying 7.5x more per call than it did at the start — it's paying 7.5x more per call, for every remaining call, for the rest of the session. Two other people have already written the sharper version of this observation ([Context Tax: Step 12 Costs 42x Step 1](https://dev.to/search?q=Context+Tax+Step+12+Costs+42x+Step+1), [the quadratic cost of long-running agents](https://blog.exe.dev/expensively-quadratic)), so I won't re-derive it here. What I hadn't seen anywhere was a *live, glanceable* number that tells you when it's actually happening to you, not just that it happens in general.

The naive fix — show cost-per-turn in the status line — doesn't work. Raw per-turn cost ÷ per-turn context growth is a ratio with a denominator that is often near zero and sometimes negative (context shrinks after compaction), so the number swings 40x between adjacent turns for no behavioral reason. A fixed time window (say, "cost over the last 10 minutes") is stable but laggy — it keeps a stale cheap turn from three minutes ago averaged into a number that's supposed to tell you what's happening *now*.

## The metric

The thing that actually works is an exponentially-weighted moving average of the numerator and denominator *separately*, not of the ratio:

```
rate = ewma(turn_cost) / ewma(net_context_growth)   [× 1e6, reported as $/1M]
```

Each new turn moves both EWMAs toward its own values with weight `alpha` (0.4 here) — so the newest turn has real influence (the number is live) while the denominator stays a smoothed aggregate rather than division-by-near-zero noise (the number isn't chaotic). This is the standard rate-smoothing pattern from systems monitoring, applied to something monitoring dashboards don't usually track: the marginal cost of an LLM agent's own memory.

Alongside the smoothed rate, the exact cost of the *last* completed turn is shown with zero smoothing — the ground truth, next to the trend.

Rendered in the status line, it looks like:

```
⛁141K  Δ$4.72  🟡$308/M
```

- `⛁141K` — live absolute context size
- `Δ$4.72` — exact cost of the last turn, no smoothing
- `🟡$308/M` — EWMA $/1M-ctx-growth, with a severity glyph

The bands (derived first-principles from a representative model's per-token pricing and cache economics — not a published benchmark, and deliberately made configurable since they're pricing-dependent):

| Band | Range | Meaning |
|---|---|---|
| 🟢 | < \$30/M | Healthy accumulation — spend tracks real growth |
| ⚪ | \$30–150/M | Normal tool use, mild resend tax |
| 🟡 | \$150–400/M | Low net growth (churn) or partial cache misses |
| 🔴 | ≥ \$400/M | Big context + cache miss + near-zero growth — branch or trim |

The peak that motivated this whole thing was \$924.67/M in a session that had ballooned to a large context and then hit a cache miss on a turn that added almost nothing — full-price resend of the whole window, for close to zero retained value. That's the number a glance at the status line should catch before it happens three more times in the same session.

## The bug that made this a real debugging story, not just a formula

The obvious way to compute "current context size" from an OMP/Pi-style agent's session state is to walk the assistant message history and sum two fields every message carries:

```ts
message.contextSnapshot = { promptTokens, nonMessageTokens }
```

`nonMessageTokens` looks like exactly what its name says: tokens *outside* the message history — system prompt, tool definitions, injected memory. The natural instinct is `total = promptTokens + nonMessageTokens`. I built it that way first. It showed 187k on a session OMP's own native footer reported as 141k.

Tracing it across every turn of a live session confirmed the actual relationship: `promptTokens === usage.input + usage.cacheRead + usage.cacheWrite` exactly, on every single turn. `promptTokens` **is** the full billed context for that turn. `nonMessageTokens` (a near-constant ~45k — system prompt + tool schemas + memory, in this setup) is a *subset* of `promptTokens`, not an addend. Adding them double-counts the constant term.

The interesting part is why this bug is invisible in the sibling extension it was copied from. That extension only ever computes *deltas between turns* (`this_turn_ctx - previous_turn_ctx`) to drive a cost-per-growth ratio. A near-constant offset cancels perfectly in a delta — `(promptTokens_a + K) - (promptTokens_b + K) = promptTokens_a - promptTokens_b`. So the formula is *correct* for the thing it's used for there, and *wrong* the moment you reuse it to render an absolute gauge instead of a difference. Same code, same field names, one consumer where the bug can't show and one where it's off by 33%. The fix was one line: drop `+ nonMessageTokens`, use `promptTokens` alone — which is also exactly what the platform's own built-in context-percentage indicator does.

Lesson, generalized past this one extension: a formula validated against *one* use of a shared value is not validated against every use of that value. If you lift a computation for a new purpose, re-derive from the field semantics, don't just trust that it already worked somewhere else.

## The counter-intuitive part

After restarting a session (resume, not fresh start), the rate *drops* — even though the agent is, in some naive sense, "carrying the same context" it had before. This looks backwards until you remember what the metric actually measures: growth-*efficiency*, not context volume. A resumed session's initial context arrives via cache reads, which are nearly free and add ~zero *net new* tokens relative to what was already established. The denominator (growth) stays near zero and the numerator (cost) stays low too — cheap turn, little growth, unremarkable ratio. The number was never telling you how much context you're carrying (that's what the absolute `⛁141K` gauge is for); it's telling you how expensively you're currently *adding to* that pile. Those are different questions, and conflating them is exactly the trap the naive "show me cost per turn" version falls into.

## Design notes

- **Standalone, not a patch to existing code.** This lives as its own extension rather than modifying the retro tool it borrows the branch-walking pattern from — that tool was stable and I didn't want a display feature risking a regression in something unrelated. It replays the session's message history read-only, so it's stateless: no baseline to seed, safe across restarts and resumes by construction.
- **Configurable bands, not hardcoded.** Cost thresholds are a function of model pricing, which changes. Hardcoding \$30/\$150/\$400 as constants would have quietly gone wrong the next time the underlying model's per-token cost shifted. They're read from a small config file, with strict validation and a safe fallback to the derived defaults on anything malformed.
- **Minimal display, no verbs.** The first version had color-coded action hints ("consider branching"). Cut in favor of just the number and a glyph — the point was a glanceable gauge, not another thing telling you what to do.

## Source

- Public prior art on the underlying cost curve: [Expensively Quadratic](https://blog.exe.dev/expensively-quadratic), ["Context Tax: Step 12 Costs 42x Step 1"](https://dev.to/search?q=Context+Tax+Step+12+Costs+42x+Step+1), [tianpan.co on quadratic agent cost](https://tianpan.co)
- Earlier post in this series on the same context-cost-visibility thread: {% post_url 2026-07-20-omp-pause-extension-and-feature-discovery %}
</content>
</invoke>
