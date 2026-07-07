---
layout: post
title: "The next OMP + Sonnet failures: replay, legacy thinking shapes, and poisoned sessions"
date: 2026-07-07
categories: ai agents productivity
series: "AI coding agent productivity"
---

A few days ago I wrote about one specific OMP turn-2 replay failure and the narrower fix that kept reasoning enabled. That post was not the end of the story.

The next few sessions failed in new ways: first the original turn-2 replay problem, then a Sonnet 5 request-shape regression, then a blank-signature replay-poison bug in an already-live session. From the UI they all looked like "Sonnet is broken again". From the raw evidence they were three different incidents.

This post is the sanitized sequel: symptom, evidence, fix. No private endpoints, no provider secrets, no company-specific infrastructure.

---

## The incident table

| Symptom | Evidence to look for | Fix that held |
|---|---|---|
| Turn 1 works, turn 2 fails in the same session or after `--resume` | Same model looks healthy on turn 1 but continuation breaks on turn 2 | Switch that model to a non-adaptive effort-based thinking mode and verify on a real turn-2 replay |
| `Model "..." not found` appears right after editing thinking config | This starts immediately after a model config edit, especially around thinking levels | Remove invalid `thinking.efforts` values; `off` does not belong there |
| Sonnet 5 starts 400ing and the raw request shows `thinking: { type: "enabled", budget_tokens: ... }` | Request shape itself is wrong before replay even enters the picture | Put Sonnet 5 back on adaptive thinking |
| Sonnet 5 starts 400ing with `Invalid \`signature\` in \`thinking\` block` and replayed assistant history contains `thinking.signature: ""` | The current session is replaying poisoned assistant history, often alongside prior tool-use blocks | Protect future sessions by disabling unsigned-thinking replay for that provider path, and restart in a fresh session for the already-poisoned one |
| Sonnet 5 fails only on a strict structured-output / tool path | Raw 400 names structured outputs as unsupported | Disable strict-tools behavior for that route or use a build that already does |

The important shift is from **symptom → evidence → fix**, not symptom → guess → random config edits.

---

## 1) First came the turn-2 replay bug

The first failure was seductive because it looked fixed almost immediately.

A fresh session on Sonnet 4.6 worked. A simple prompt worked. Even a config tweak appeared to help. But the real bug only showed up on **turn 2** — sometimes in the same session, sometimes after `--resume`.

The working fix was not "turn thinking off everywhere". That was only the emergency unblocker.

The better fix was model-local and narrower:

```yaml
thinking:
  mode: anthropic-budget-effort
  efforts: [low, medium, high]
  defaultLevel: low
```

That preserved reasoning while avoiding the adaptive replay path that was breaking continuation.

The verification rule that mattered was stricter than "turn 2 returned text":

1. turn 2 must return text
2. it must stay on the intended provider/model
3. it must still carry a thinking block if reasoning is supposed to remain enabled

Anything weaker is how you fool yourself.

---

## 2) The fake loader bug: `Model not found` was really schema drift

This one wasted more time than it deserved because the error message pointed in the wrong direction.

After editing model config, OMP started reporting that the custom model could not be found. That smelled like profile drift, catalog load failure, or an override-file problem.

It was actually just this mistake:

```yaml
thinking:
  efforts: [off, low, medium, high]
```

In this schema, `off` is **not** a valid effort. The accepted values are:

- `minimal`
- `low`
- `medium`
- `high`
- `xhigh`

So the real fix was not anything architectural. It was removing the invalid enum value.

That is an important category of failure in agent tooling: a config validation error surfacing as a runtime resolution error. If you are not careful, you go looking for bugs in the wrong subsystem.

---

## 3) Then Sonnet 5 had two different 400s pretending to be one

The most frustrating part of the last few days was that "Sonnet 5 400" was not one issue. It was two.

### Mode A: legacy manual-thinking request shape

The raw request showed a thinking block like this:

```json
{
  "thinking": {
    "type": "enabled",
    "budget_tokens": 2048
  }
}
```

That turned out to be local override drift. The model had been pushed onto an older manual-thinking dialect when this path wanted adaptive thinking.

The fix was to stop forcing the old manual mode and put Sonnet 5 back on adaptive thinking.

### Mode B: replay-poisoned blank signatures

A different 400 looked like this instead:

```text
invalid_request_error: messages.1.content.0: Invalid `signature` in `thinking` block
```

The corresponding replayed assistant history contained:

```json
{
  "type": "thinking",
  "thinking": "...",
  "signature": ""
}
```

That is not a first-turn config-shape problem. That is a replay-history problem.

The remedy is also different:

- protect future sessions by disabling unsigned-thinking replay for that provider path
- **restart in a fresh session** when the current one is already replay-poisoned

That distinction matters because otherwise you keep editing config for a session that is already dead.

---

## 4) The current session may be doomed even when the config is fixed

This was the operator lesson I needed most.

A config fix can be real and still not save the session you are staring at.

Once a session has already stored assistant replay history with blank thinking signatures, the next request may keep replaying that bad history. At that point, saying "the bug is fixed" is sloppy. What is actually true is narrower:

- **future fresh sessions are protected**
- **this live session may still be poisoned**

Those are different claims.

In my case, the only verified remedy for the poisoned session was: **start a fresh session**.

That sounds obvious in retrospect. It was not obvious while looking at a UI that kept presenting every failure as just another generic 400.

---

## 5) Strict tools was a separate trap again

There was another Sonnet 5 failure class that had nothing to do with replayed signatures.

This one failed on the strict-tools / structured-output path. The model route itself did not support that dialect, so the fix was not about replay, thinking mode, or session history. The fix was to disable strict-tools behavior for that route — or move to a build where the route detection already does that.

This is another good example of why one label like "Sonnet 5 failure" is too coarse to debug from.

The shortest useful triage question is not:

> Did Sonnet fail?

It is:

> What exactly did the raw 400 say?

---

## The decision tree I wish I had on day one

If I hit this family of bugs again, I would do exactly this:

1. **Capture the raw 400 first**
   - no UI-only diagnosis
   - no guessing from memory
2. **Classify the failure by the error text / request shape**
   - `type: "enabled" + budget_tokens` → manual-thinking drift
   - `Invalid \`signature\` in \`thinking\` block` → replay-poisoned session history
   - `structured_outputs not supported` → strict-tools route mismatch
3. **Apply the narrowest fix for that class**
4. **Re-test on the smallest experiment that proves the assumption**
   - usually a fresh session and a real turn-2 replay
5. **Do not keep using a poisoned session just because config has improved**

That last rule alone would have saved a fair amount of frustration.

---

## What these failures had in common

All of them were really the same meta-problem:

**state leaked across layers**.

- model config state
- provider compat state
- replayed session state
- tool/structured-output capability state

From the terminal, they all looked like "Claude is broken again".

But each failure lived at a different layer, and the correct fix only became obvious once I identified which layer was actually lying.

That is why I now think agent debugging needs two separate habits:

1. **raw-error-first discipline**
2. **fresh-session verification after any replay-related fix**

Without those two habits, you can spend an afternoon fixing the previous bug while the current bug keeps winning.

---

## The stable takeaways

- Never trust a one-turn success on a multi-turn failure.
- Never trust a generic 400 label when the raw error is available.
- Never assume a current session is recoverable just because future sessions are now configured correctly.
- Never collapse request-shape bugs, replay-history bugs, and capability-route bugs into one bucket.

For the last few days, the frustration came mostly from violating one of those rules at a time.

Now that they are written down, I expect the next round to be a great deal less dramatic.

## Source

- [OMP issue #4060](https://github.com/can1357/oh-my-pi/issues/4060)
- [OMP issue #4297](https://github.com/can1357/oh-my-pi/issues/4297)
- [OMP issue #4679](https://github.com/can1357/oh-my-pi/issues/4679)