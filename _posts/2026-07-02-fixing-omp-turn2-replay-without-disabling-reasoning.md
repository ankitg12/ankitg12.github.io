---
layout: post
title: "Fixing OMP turn-2 replay failures without disabling reasoning"
date: 2026-07-02
categories: ai agents productivity
series: "AI coding agent productivity"
---

One fine Wednesday morning I updated OMP and my Claude Sonnet 4.6 (`claude-sonnet-4-6`) sessions stopped working — this is a story of getting them back up again.

Turn 1 still looked fine, which made the bug easy to misdiagnose. The real failure showed up on turn 2: sometimes in the same live session, sometimes after `--resume`, and sometimes as a silent fallback to another model if I was not watching the JSONL closely.

I filed the upstream request as [OMP issue #4060](https://github.com/can1357/oh-my-pi/issues/4060) once it was clear this was about adaptive-thinking replay, not just a bad one-off session.

The first workaround I found was blunt: disable thinking globally. It worked, but it also degraded every model in OMP. The better result came later: a **model-local** config that kept reasoning enabled, switched away from the problematic adaptive path, and still survived turn 2.

This post documents that progression, the traps on the way, and the exact verification pattern that kept me from calling it fixed too early.

---

## The problem

The failure mode was specific:

1. Start a fresh OMP session on Claude Sonnet 4.6 (`claude-sonnet-4-6`) through the affected Anthropic-compatible proxy
2. Ask a simple prompt
3. Ask a second prompt in the same live session — or resume the session and ask it there
4. Watch turn 2 fail or silently fall back to another model

That last part matters. If your config has fallback chains, turn 2 can appear to "work" even when the target model actually failed and OMP switched providers behind your back.

So the acceptance criterion is **not** "turn 2 returned text". The acceptance criterion is:

- turn 2 returned text
- turn 2 still ran on the intended provider/model
- if reasoning is meant to stay enabled, turn 2 still carried a reasoning/thinking block

Anything weaker is a contaminated test.

---

## The first unblocker: global thinking off

The fastest supported OMP workaround was:

```yaml
defaultThinkingLevel: off
omitThinking: true
```

That unblocked turn 2 immediately.

It also had two drawbacks:

1. It is **global** — it changes behavior for all models, not just the affected one
2. It throws away explicit reasoning mode entirely

That is still a useful emergency lever. If you just need the model usable again, this is a valid first move.

But it is not the best steady state.

---

## The better fix: model-level non-adaptive reasoning

The stronger workaround lives in `models.yml`, on the affected model entry itself.

```yaml
- id: claude-sonnet-4-6
  name: Claude Sonnet 4.6
  reasoning: true
  thinking:
    mode: anthropic-budget-effort
    efforts: [low, medium, high]
    defaultLevel: low
```

Then remove the earlier global off-switch from `config.yml`.

What this did in practice:

- kept reasoning **enabled**
- limited the workaround to **one model** instead of all OMP models
- avoided the turn-2 replay failure
- preserved more capability than `thinking off`

This is the important distinction: the final fix was **not** "turn thinking off for the affected model". It was "use a non-adaptive effort-based mode for that model so replay survives turn 2".

---

## The verification pattern

A proper repro needs a real second turn on Claude Sonnet 4.6 (`claude-sonnet-4-6`), not just turn 1. In my testing the failure could appear either on turn 2 of the same live session or on turn 2 after `--resume`. The resumed form was easier to script and inspect, so this is the concrete variant I used:

```bash
omp --session-dir ~/tmp/turn2-repro --model amd-claude/claude-sonnet-4-6 --mode json -p "Reply with exactly: turn1-ok"
omp --session-dir ~/tmp/turn2-repro --resume <session-id> --mode json -p "Reply with exactly: turn2-ok"
```

Then inspect the JSONL for four things:

1. a `model_change` event showing the intended selector
2. a `thinking_level_change` event showing the configured model-level default (for me: `low`)
3. turn 1 assistant message still on the intended provider/model
4. turn 2 assistant message still on the same provider/model

If you are trying to preserve reasoning, add a fifth check:

5. both assistant messages still include a reasoning/thinking block

That last check is what upgraded this from a blunt workaround to a real fix.

---

## What made this annoying to debug

### 1. One-turn tests lie

Turn 1 was never the real problem. The bug showed up on the second turn — in the same live session or after OMP resumed prior reasoning.

If you stop at a single prompt, you can easily declare victory on a broken config.

### 2. "Model not found" was a misleading symptom

At one point OMP stopped recognizing my custom selector entirely and reported:

```text
Model "..." not found
```

That looked like a loader or profile problem.

It was actually a config-shape bug in my new thinking block.

I had tried this:

```yaml
thinking:
  mode: <non-adaptive effort mode>
  efforts: [off, low, medium, high]
  defaultLevel: low
```

That is invalid. In OMP's model schema, `thinking.efforts` accepts:

- `minimal`
- `low`
- `medium`
- `high`
- `xhigh`

`off` is **not** one of them.

Once I changed it to:

```yaml
thinking:
  mode: <non-adaptive effort mode>
  efforts: [low, medium, high]
  defaultLevel: low
```

model discovery came back immediately.

This was the hardest part of the session to explain cleanly: a model-level config validation mistake surfaced as what looked like a runtime model-resolution failure.

### 3. Tiny overlay configs were unreliable for this repro

I initially tried a minimal `--config` file just to disable fallback for testing. In practice that made OMP behave as if it could not see my custom models at all.

The safer path was:

- test against the **real** config
- then verify the turn-2 JSONL still shows the same provider/model

That is slower than forcing fallback off in a tiny overlay, but it is more trustworthy.

---

## Cost implications of reasoning levels

There are two separate questions here:

1. **Does this preserve capability?** — yes, more than the global off-switch
2. **Does this cost more?** — also yes, generally

From the model provider's public docs:

- reasoning tokens are billed as **output tokens**
- prior reasoning can also become **later-turn input cost** when replayed in context
- omitting reasoning output affects visibility/latency, **not** billing

So the rough cost ordering is:

| Setting | Capability | Cost |
|---|---|---|
| `thinking off` | lowest | cheapest |
| `low` | higher | higher |
| `medium` | higher again | higher again |
| `high` | highest of these | usually highest |

That is why `low` was the right first test. It preserves explicit reasoning, but keeps the blast radius on both latency and cost smaller than `medium` or `high`.

One more nuance: on newer reasoning-model behavior, older manual budget-style controls are being deprecated in some provider docs. So I would treat the non-adaptive effort mode here as a **working local compatibility workaround**, not a timeless universal best practice.

---

## What I would do next time

If I hit this again, I would use this order immediately:

1. **Confirm the bug with a real second turn**
   - same live session or resumed session, but never trust turn 1 alone
2. **Use the global off-switch only as an emergency unblocker**
   - `defaultThinkingLevel: off`
   - `omitThinking: true`
3. **Move to the narrower model-local fix for Claude Sonnet 4.6**
   - switch from adaptive reasoning to a non-adaptive effort-based mode
   - start with `defaultLevel: low`
4. **Verify turn 2 stayed on the intended model and still carried reasoning**
5. **Only then** think about whether the runner itself should grow a cleaner provider-scoped setting, as discussed in [OMP issue #4060](https://github.com/can1357/oh-my-pi/issues/4060)

That ordering would have saved me a lot of tokens.

---

## Why this matters beyond this one bug

The general lesson is bigger than one runner or one provider.

When an agent stack fails on a multi-turn feature, you need to separate three different goals:

- **unblock usage now**
- **preserve capability if possible**
- **find the clean upstream fix later**

Those are not the same problem.

The global `thinking off` workaround solved the first. The model-local non-adaptive reasoning config solved the second. An upstream provider-scoped fix would solve the third.

If you collapse them into one question, you either patch too early or settle for a workaround that is blunter than necessary.

---

## Final config shape I would keep

```yaml
- id: claude-sonnet-4-6
  name: Claude Sonnet 4.6
  reasoning: true
  thinking:
    mode: anthropic-budget-effort
    efforts: [low, medium, high]
    defaultLevel: low
```

And remove this from your global config once the narrower fix is proven:

```yaml
defaultThinkingLevel: off
omitThinking: true
```

---

## Source

- [OMP issue #4060](https://github.com/can1357/oh-my-pi/issues/4060)
- local reproduction notes for Claude Sonnet 4.6 (`claude-sonnet-4-6`)
- runner config schema for model-level thinking
- public provider docs on reasoning token billing and effort controls
