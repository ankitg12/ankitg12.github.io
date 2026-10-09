---
layout: post
title: "A Meter for What /shake Would Free"
date: 2026-10-09 13:30:00 +0530
categories: ai agents productivity
series: "AI coding agent productivity"
---

For three months I typed `/shake` into my coding agent every five to ten minutes, and every time I
learned what it was worth only after I had paid for it. Sometimes it freed 2,000 tokens. Sometimes
it freed 30,000. Nothing on screen told me which it would be.

`/shake` is [oh-my-pi](https://github.com/can1357/oh-my-pi)'s (OMP) mechanical context cleaner. It
replaces old, large tool results and long fenced blocks with a one-line marker such as
`[shaken ~1234 tokens — recover: artifact://62 (region 12)]`, and keeps the removed text in a
recovery file. No LLM call, no summary, nothing lost for good. It is cheap to run, but not free:
it rewrites earlier context, so the next request misses the provider's prompt cache. Running it for
2k tokens is a bad trade. Running it for 30k is a good one.

So I built a footer item that answers one question before I act:

```
 ~27k shakeable
```

## The gap: usage is not reclaimability

Don Norman called this the *gulf of evaluation*: the distance between what a system is doing and
what its user can perceive about it ([NN/g summary](https://www.nngroup.com/articles/two-ux-gulfs-evaluation-execution/),
from *The Design of Everyday Things*, 1988). Every agent I use shows how *full* the context window
is. None shows how much of it is *dead weight* that a cleanup would remove. A full window with
nothing reclaimable calls for a summary or a new session. A full window that is 40% stale tool
output calls for `/shake`. A usage bar cannot tell the two apart.

What I found when I looked on 2026-10-09:

| Tool | What its footer shows | Answers "what would a cleanup free now?" |
| --- | --- | --- |
| OMP built-in footer | Context usage | No |
| [pi-context-usage](https://github.com/championswimmer/pi-context-usage) | Dot-grid of context usage | No |
| [pi-context-prune](https://github.com/championswimmer/pi-context-prune) | Prune on/off, cumulative summarizer tokens and cost | No — it reports what pruning *cost*, after the fact |
| `shake-meter-omp` (this post) | Tokens the next `/shake` would free | Yes |

I do not claim nobody has built this. I claim I could not find it. If you know of one, tell me.

## The meter

`shake-meter-omp` is a display-only OMP extension. It never changes the session.

- **Number:** tokens a no-argument `/shake` would free from the current branch, right now.
- **Colour:** dim below 10k, the theme's warning colour from 10k, its error colour from 25k. At a
  glance: ignore, consider, do it.
- **Refresh:** on session start, switch, compaction, and every turn end. `/shake` itself emits no
  extension event, so a 3-second tick is what makes the number drop after you shake.
- **Log:** every change goes to `~/.omp/logs/shake-meter.jsonl`, so the meter can be checked
  against what `/shake` later reports.

It is deliberately separate from the extension that *acts* on context (my
[session governor]({% post_url 2026-10-06-session-governor-one-policy-engine-for-the-whole-agent-session %})). A gauge
should not be part of the engine it measures. The governor imports the same function as a rule
variable, so the footer and the rule always show one number:

```yaml
# ~/.omp/governor.yml
- name: shakeable-prune
  when: 'shakeable > 25000 && turns_since_prune >= 2'
  prune: true
```

`turns_since_prune >= 2` keeps the rule from busting the prompt cache on every turn.

## Getting the number right

The first version was wrong by half. The footer said about 15k; `/shake` reported
`Shook 43 tool results + 1 block (~27838 tokens freed)`.

The rules were right: which results are eligible, the 4,000-token protect window at the end, the
skip list for skills and already-shaken results, the 400-token minimum for fenced blocks. I had
restated them from OMP's source in my own code, because I assumed an extension could not import
them from the compiled binary. The *unit* was wrong. I counted tokens as characters ÷ 4, the
familiar rule of thumb for English prose. OMP counts with the model's real tokenizer, and on this
session's tool output Claude's tokenizer measured about 2.2 characters per token. Code, JSON, paths
and hashes are dense.

The fix was to stop restating the host and call it. Inside OMP, a bare import of
`@oh-my-pi/pi-agent-core` resolves to the running agent's own module. I learned that the hard way in
{% post_url 2026-08-15-reusing-an-agents-internal-ui-from-an-extension %}. That module exports the
same three things `/shake` uses:

```ts
const { collectShakeRegions, AGGRESSIVE_SHAKE_CONFIG, Tokenizer } = await import("@oh-my-pi/pi-agent-core");
const tokenizer = new Tokenizer(ctx.model);
const regions = collectShakeRegions(branch, tokenizer, AGGRESSIVE_SHAKE_CONFIG);
const freed = regions.reduce((sum, r) => sum + r.tokens - placeholderTokens, 0);
```

The import is dynamic on purpose: outside OMP (unit tests, a future rename) it fails, and the meter
falls back to the chars/4 restatement and logs `"source":"estimate"`, so a wrong number is never
silent.

## Proof: replay real shakes

Each `/shake` leaves a recovery file with one header per region, e.g.
`### region 12 (read, ~412 tok)`. That is enough to rebuild the session as it was a moment before
the shake and run the meter on it. `verify-shake.ts` does exactly that. Four shakes from one session:

| Shake | `/shake` reported | Meter now | Meter v1 (chars/4) |
| --- | --- | --- | --- |
| 1 | ~2,469 freed | 2,460 | 1,254 |
| 2 | (not captured) | 15,624 | 7,750 |
| 3 | ~27,838 freed | 27,237 | 14,614 |
| 4 | (not captured) | 33,304 | 15,938 |

The remaining 2% on shake 3 comes from the rebuild, not the meter: the recovery file holds the
removed text, not the original message byte for byte, and the rebuild found 43 regions where
`/shake` found 44. On shake 4 the region counts match exactly, 63 and 63. After the fix, the live
log reads:

```
{"tokens":6139,"toolResults":10,"blocks":0,"source":"omp"}
```

## The road not taken

An extension still cannot *run* `/shake`; OMP passes `sendUserMessage("/shake")` to the model as
plain text. For an afternoon I had the governor type `/shake` into its own terminal pane through
the terminal multiplexer, after waiting for the agent to go idle and checking the prompt editor was
empty. It worked on paper and I deleted it: two transports to maintain, a race if I started typing
at the wrong moment, and all of it to be thrown away the day the real API lands. An out-of-band
*key* is fine (my [attention pause]({% post_url 2026-08-14-building-attention-pause-for-herdr-and-omp %})
sends one); out-of-band *typing* into a human's editor is not.

The real fix is upstream: [oh-my-pi#5661](https://github.com/can1357/oh-my-pi/issues/5661) asks for
`ExtensionContext.shake(mode)`. With that, the meter tells you when, and the governor does it.

## Source

- [`shake-meter-omp`](https://github.com/ankitg12/omp-extensions/tree/main/packages/shake-meter-omp) — the meter, `preview.ts`, `verify-shake.ts`
- [`session-governor-omp`](https://github.com/ankitg12/omp-extensions/tree/main/packages/session-governor-omp) — the `shakeable` rule variable
- [oh-my-pi](https://github.com/can1357/oh-my-pi) — `packages/agent/src/compaction/shake.ts` holds the rules
- [oh-my-pi#5661](https://github.com/can1357/oh-my-pi/issues/5661) — extension-side `shake()`
