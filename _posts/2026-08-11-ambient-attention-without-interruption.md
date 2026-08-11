---
layout: post
title: "Ambient Attention Without Interruption"
date: 2026-08-11
categories: ai agents productivity terminal
series: "AI coding agent productivity"
---

A notification that makes me open Slack has already taken more attention than it saved.

The problem appears simple: I spend long stretches inside terminal coding agents and do not want to miss a direct message or mention. The obvious solutions are all slightly wrong:

- native desktop banners interrupt the work;
- notification sounds create anxiety without context;
- a Slack TUI puts the entire distraction inside the terminal;
- periodic manual checking guarantees context switches;
- injecting messages into the agent conversation wastes model context and changes what the model sees.

What I needed was not another notification system. I needed **ambient assurance**: a quiet, persistent signal that tells me whether something genuinely new is waiting, without showing the message or asking me to act immediately.

## The interface is a three-state contract

The final interface is deliberately boring:

```text
Slack ✓     collector healthy; no new attention since baseline
Slack +1    one newly attentive conversation remains unread
Slack !     collector failed or its state can no longer be trusted
```

No sound. No toast. No sender name. No message preview.

The distinction matters. An absolute unread count encourages checking because it exposes backlog. A positive delta answers the narrower question: **has anything new arrived since I started working?**

The signal persists until the corresponding conversation is no longer reported unread. It therefore survives switching between terminal agents without becoming an ephemeral popup that can be missed.

## One source, many quiet consumers

The architecture separates collection from presentation:

```text
                 one collector
                      │
                      │ sanitized atomic write
                      ▼
          slack-attention.json
             │                │
             │                │
      OMP status bar     Herdr Space metadata
      in every agent     outside agent context
```

The collector is global. Agent sessions do not contact Slack. Each session observes the same small state file and renders only the status token.

The state contains identifiers and counters needed for delta detection, but no credentials, message text, sender names, or channel names. The first successful sample establishes a quiet baseline. Later samples add newly attentive conversation identifiers to a persistent set; an identifier leaves the set only when the conversation is no longer unread.

This avoids two common agent-integration errors:

1. **one poller per agent**, which duplicates calls and can produce conflicting baselines;
2. **message injection**, which turns an operational signal into model input.

## The detours were the useful part

The implementation was not a straight line. Each wrong turn refined the actual requirement.

### Detour 1: fixing Windows toasts

The first hypothesis was transport failure. A notification produced sound but no visible banner, so I inspected Windows notification registration, focus settings, and application identifiers. A control toast emitted through a registered PowerShell application appeared correctly.

That proved Windows could display banners. It did not prove banners were the right product.

The investigation had optimized the transport before defining the desired interaction. Even a perfectly delivered Slack banner would still interrupt the terminal session and invite an immediate context switch.

**Lesson:** verify the human requirement before repairing the delivery mechanism.

### Detour 2: trusting `shown: true`

A direct Herdr notification call returned a successful result equivalent to:

```json
{"shown": true, "reason": "shown"}
```

No human-visible toast appeared.

The result proved that the server accepted the request. It did not prove that the foreground client rendered it. API acknowledgement and human-visible delivery are different observations; treating one as the other creates a false success claim.

**Lesson:** UI delivery needs human-visible or screen-captured evidence. A successful return code is insufficient.

### Detour 3: showing absolute unread counts

The first persistent badge displayed the current unread level and accumulated mention counters. It worked technically and failed psychologically.

Standing counters mix old backlog with new attention. Large historical values create pressure to inspect Slack even when nothing new has arrived. The display was correct as telemetry and wrong as an interface.

The replacement tracks only positive deltas after baseline:

```text
historical backlog at startup  → Slack ✓
new unread conversation        → Slack +1
conversation becomes read      → Slack ✓
```

**Lesson:** operational telemetry and attention interfaces optimize for different things.

### Detour 4: considering a Slack TUI

A dedicated Slack terminal pane sounds attractive because it avoids switching to a desktop window. It also brings channels, message bodies, threads, and replies into the workspace where focus is supposed to happen.

That would move the distraction rather than remove it. It would also introduce another authenticated client, another unread model, and another component to maintain.

**Lesson:** proximity is not the same as low friction. Sometimes the best integration exposes less capability.

### Detour 5: missing the first atomic file creation

The status appeared when refreshed manually and updated on later writes, yet two fresh agent sessions both missed the initial state-file creation.

The writer used the correct durability pattern:

1. write a temporary file;
2. rename it atomically over the destination.

The consumer watched the parent directory but filtered events to the final filename. On Windows, the watcher first reported the temporary side of the rename sequence. Both fresh consumers ignored it, so neither loaded the baseline.

The fix was to accept temporary and final state filenames and debounce the event burst once before rereading the authoritative destination:

```ts
const stateName = basename(statePath);

watch(directory, (_event, filename) => {
  const changed = filename?.toString();
  if (
    changed &&
    changed !== stateName &&
    !changed.startsWith(`${stateName}.`)
  ) return;

  clearTimeout(reloadTimer);
  reloadTimer = setTimeout(readAuthoritativeState, 25);
});
```

This is not polling. It is a one-shot delay that lets the atomic rename settle.

The corrected version was tested with two genuinely fresh OMP processes:

| Transition | Session A | Session B |
|---|---:|---:|
| Initial atomic create | `Slack ✓` | `Slack ✓` |
| One new conversation | `Slack +1` | `Slack +1` |
| Conversation removed from unread set | `Slack ✓` | `Slack ✓` |

**Lesson:** a manual refresh is not evidence that an event-driven integration works. Test cold-start creation in multiple fresh consumers.

### Detour 6: green without a heartbeat

The final conceptual bug was subtler. If no recurring collector is running, the last successful state can remain green forever. A stopped watcher and a quiet Slack account become visually indistinguishable.

A green health indicator is therefore not a boolean. It is a **lease**:

```text
healthy = last_success + allowed_age > now
```

The collector must renew that lease. Herdr metadata should carry a time-to-live, and local consumers should mark the state failed after the renewal deadline. If recurring collection is not active, the honest interface is no badge at all—not remembered success.

The implementation was deliberately left dormant after testing: no state file, no badge, and no background collector. Activation requires both an authorized schedule and explicit stale-state semantics.

**Lesson:** silence is trustworthy only when liveness is observable.

## Verification gates

The useful test suite is not “did the command exit zero?” It covers the contract:

- credentials never survive the collector boundary;
- first sample establishes baseline without attention;
- new conversation produces one persistent delta;
- unchanged state does not duplicate attention;
- decreasing or disappearing state clears attention;
- malformed collection cannot overwrite the last valid state;
- collector failure renders an unhealthy state;
- old state migrates without turning backlog into new attention;
- two fresh agent sessions observe the same cold-start create, delta, and clear sequence.

The repository-level tests and the live fixture test serve different purposes. Unit tests prove state transitions. Fresh terminal sessions prove that the operating system, atomic writer, file watcher, extension loader, and UI status path compose correctly.

## The general pattern

This design is not specific to Slack. It applies whenever a developer needs awareness without interruption:

- build or deployment health;
- incident escalation;
- code-review requests;
- calendar urgency;
- long-running experiment completion.

The pattern is:

1. collect once globally;
2. sanitize before persistence;
3. derive attention from positive deltas, not standing levels;
4. render outside model context;
5. persist until the underlying condition clears;
6. expose liveness through a renewable lease;
7. prove cold-start behavior in multiple fresh consumers.

The deeper principle is simple: **good attention tooling raises awareness without raising activity.**

## Source

- [Oh My Pi — terminal coding agent](https://github.com/can1357/oh-my-pi)
- [Herdr — runtime for coding agents](https://github.com/herdrdev/herdr)
- [Node.js file-system watcher documentation](https://nodejs.org/api/fs.html#fswatchfilename-options-listener)
- [Slack notification configuration](https://slack.com/help/articles/201355156-Configure-your-Slack-notifications)
