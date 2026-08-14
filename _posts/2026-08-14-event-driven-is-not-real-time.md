---
layout: post
title: "Building Attention Pause for Herdr and OMP"
date: 2026-08-14
categories: ai agents productivity terminal
series: "AI coding agent productivity"
---

Two OMP agents were running in separate Herdr panes, but changing focus did not change which agent could continue working.

[Herdr](https://github.com/herdrdev/herdr) is a terminal multiplexer built for coding agents. [Oh My Pi (OMP)](https://github.com/can1357/oh-my-pi) is a terminal coding agent with a native `/pause` command. Together, they expose the pieces for an attention policy:

> Only the OMP agent in the currently focused Herdr pane may run. Every background OMP agent should enter its native pause state and remain there until I resume it manually.

This post documents the `attention-pause` plugin I built, the real UI tests that broke its early designs, and the difference between a useful best-effort guard and a deterministic focus lock.

## Why native pause matters

The wrong implementation would stop a process, send `Ctrl-C`, or suspend the terminal pane. Those actions can interrupt a write, leave a tool call incomplete, or corrupt conversational state.

OMP already owns the correct control boundary. Its `/pause` command:

- allows an in-flight tool call to finish;
- prevents new agent work from starting;
- displays a native pause overlay;
- resumes only after manual input.

The plugin therefore does not invent another pause mechanism. It sends the command OMP already understands.

## Herdr's plugin surface

Herdr plugins are TOML manifests that bind commands to lifecycle events. The first manifest subscribed to startup, agent-status changes, and three kinds of focus changes:

```toml
[plugin]
name = "Attention Pause"
plugin_id = "ankit.attention-pause"
version = "0.1.0"
platforms = ["windows"]

[[startup]]
command = ["python", "attention_pause.py"]
platforms = ["windows"]

[[events]]
on = "workspace.focused"
command = ["python", "attention_pause.py"]
platforms = ["windows"]

[[events]]
on = "tab.focused"
command = ["python", "attention_pause.py"]
platforms = ["windows"]

[[events]]
on = "pane.focused"
command = ["python", "attention_pause.py"]
platforms = ["windows"]

[[events]]
on = "pane.agent_status_changed"
command = ["python", "attention_pause.py"]
platforms = ["windows"]
```

The hook receives event data through `HERDR_PLUGIN_EVENT_JSON`, then uses Herdr's CLI to inspect and control agents.

## The reconciliation loop

The Python handler follows five steps:

1. list Herdr agents;
2. select background OMP panes;
3. read each pane's current detection screen;
4. skip panes that already show OMP's live pause footer;
5. recheck focus and submit `/pause`.

The selection rule deliberately includes idle, working, and done OMP agents. Agent status is not the policy. Focus is.

```python
def pause_candidates(agents):
    return [
        agent
        for agent in agents
        if agent.get("agent") == "omp"
        and not agent.get("focused", False)
        and agent.get("pane_id")
    ]
```

A pane can contain old `P A U S E D` text in its scrollback, so detecting that heading alone is unsafe. The handler also requires the live pause footer:

```python
PAUSE_FOOTER = "esc interrupt"


def is_paused(herdr, pane_id):
    screen = herdr_text(herdr, "agent", "read", pane_id, "--source", "detection")
    nonempty_lines = [line.strip() for line in screen.splitlines() if line.strip()]
    return bool(nonempty_lines) and nonempty_lines[-1] == PAUSE_FOOTER
```

Before submitting the command, the handler reads focus again:

```python
if is_focused(herdr, pane_id):
    print(f"now-focused pane={pane_id}")
    continue

herdr_json(herdr, "agent", "prompt", pane_id, "/pause")
print(f"pause-sent pane={pane_id}")
```

That last check closes one time-of-check to time-of-use race: a pane that became focused after the initial snapshot must not be paused.

## Unit tests passed before the interaction did

The test suite covered the policy table:

| Pane state | Focused? | Action |
|---|---:|---|
| working | yes | skip |
| working | no | pause |
| idle | no | pause |
| already paused | no | skip |
| non-OMP pane | no | skip |
| becomes focused after snapshot | yes at recheck | skip |

It also covered stale pause text, unknown status values, failed screen reads, lock contention, and a focus event arriving during reconciliation.

The suite passed. Python syntax compilation passed. Herdr reported the plugin as linked and enabled.

Then real switching failed.

## One UI switch emitted three focus hooks

Switching between two Herdr workspaces emitted:

```text
workspace.focused
tab.focused
pane.focused
```

Each event launched a separate Python process. Every process then made several command-line round trips back to Herdr:

```text
focus event
    ↓
start Python
    ↓
herdr agent list
    ↓
herdr agent read <pane>
    ↓
herdr agent list          # focus recheck
    ↓
herdr agent prompt <pane> /pause
```

The event was immediate. The control path was not. Individual handlers took several seconds, which was enough time to switch focus again.

This produced three distinct outcomes:

```text
pause-sent pane=w1F:p1
already-paused pane=w1G:p1
now-focused pane=w1F:p1
```

All three logs are locally correct. Only the first changes agent state. A late `now-focused` result means the policy missed the interval during which that pane was in the background.

## The lock detour

The first response to duplicate handlers was a non-blocking file lock. One handler reconciled; overlapping handlers returned `coalesced`.

That dropped the newest focus change. If an event arrived while reconciliation was active, no final pass was guaranteed.

The next version made waiters acquire the lock eventually. This preserved events but created a stale queue:

```text
event A ── reconcile old focus
          event B ── wait ── reconcile older focus
                    event C ── wait ── reconcile stale focus
```

The pause sometimes appeared tens of seconds later. That was not OMP waiting for an active tool call. Both agents were idle. The delay came from queued plugin handlers.

This is the important distinction:

| Delay | Meaning |
|---|---|
| Handler has not logged `pause-sent` | The plugin has not submitted `/pause` |
| `pause-sent` logged; tool still finishing | OMP is reaching its safe boundary |
| Pause appears after several stale handlers | Plugin queue latency, not agent safety latency |

Serial execution can preserve every event and still violate current-state correctness.

## Unique markers and trailing-edge reconciliation

A correct coalescer needs two properties:

1. **single flight** — only one expensive reconciliation runs at a time;
2. **trailing edge** — an event arriving during that run guarantees one final pass over the newest state.

A single dirty file was insufficient. This sequence loses an event:

```text
handler A: sees dirty file
handler B: touches the same existing file
handler A: deletes file
```

Handler B's signal disappears with handler A's delete.

Unique marker files avoid that overwrite race. A claimant snapshots and removes only the markers it observed; a marker created during deletion remains for the next waiter.

```python
def mark_dirty():
    marker = tempfile.NamedTemporaryFile(
        prefix="attention-pause.dirty.",
        dir=state_dir,
        delete=False,
    )
    marker.close()


def claim_dirty():
    markers = list(state_dir.glob("attention-pause.dirty.*"))
    for marker in markers:
        marker.unlink()
    return bool(markers)
```

This makes event coalescing lossless. It does not make Herdr CLI round trips fast.

## Reduce amplification first

Real logs proved that switching workspaces also emitted `pane.focused`. The plugin therefore removed the broader `workspace.focused` and `tab.focused` subscriptions.

The active event surface became:

```toml
[[events]]
on = "pane.focused"
command = ["python", "attention_pause.py"]
platforms = ["windows"]

[[events]]
on = "pane.agent_status_changed"
command = ["python", "attention_pause.py"]
platforms = ["windows"]
```

`pane.focused` handles human navigation. `pane.agent_status_changed` catches a background agent that becomes active without another focus change.

This reduced one switch from three focus handlers to one. It improved the system, but live tests still showed variable delay. The plugin is therefore useful as a best-effort attention guard, not as a deterministic real-time lock.

## Logs are part of the product

The handler prints one decision for every candidate:

```text
pause-sent pane=w1F:p1
already-paused pane=w1G:p1
now-focused pane=w1F:p1
read-failed pane=w1G:p1
coalesced
```

Herdr preserves each event's start time, finish time, status, stdout, and stderr:

```powershell
herdr plugin log list --plugin ankit.attention-pause --limit 60
```

Two more commands ground the UI state:

```powershell
herdr agent list
herdr agent read w1F:p1 --source detection
```

These evidence layers answer different questions:

| Evidence | Question answered |
|---|---|
| Plugin event log | Did Herdr invoke the hook? |
| Handler stdout | Why did reconciliation pause or skip? |
| Agent list | Which pane is focused now? |
| Detection screen | Is OMP's native pause overlay visible? |
| OMP session log | Did new agent work start after the boundary? |

The remaining observability gap is per-step timing inside the Python handler. Total duration cannot show whether process startup, `agent list`, screen reading, or prompt submission consumed the time.

## What deterministic would require

A real-time focus invariant should live near the authoritative focus transition:

```text
Herdr changes focused pane
    ├── records the new focus owner
    └── applies the background pause policy
```

An asynchronous plugin can observe the transition later. It cannot make its observation atomic with the transition.

The honest guarantee table is:

| Architecture | Claim |
|---|---|
| Focus owner applies policy in-process | Deterministic focus safety |
| Event plugin with asynchronous CLI calls | Best-effort delayed pause |
| Plugin disabled | No automated protection |

The next useful investigation is not another lock variant. It is whether the Herdr event payload and plugin API can remove the slow CLI discovery calls, or whether the pause policy belongs inside Herdr itself.

## Verification checklist

A final test must use the real Herdr UI:

1. Start two unpaused OMP agents.
2. Switch from agent A to agent B.
3. Record the `pane.focused` event time.
4. Confirm A receives `/pause`.
5. Return to A and confirm its native pause overlay remains visible.
6. Resume A manually.
7. Repeat rapidly enough to overlap handlers.
8. Repeat while one OMP tool call is in flight.
9. Correlate focus, submission, overlay, and session-log timestamps.
10. Reject a deterministic claim if either agent starts new work after its safe boundary.

Unit tests prove selection rules. Only this end-to-end test proves that Herdr events, subprocess startup, CLI transport, terminal state detection, and OMP control compose into the intended attention boundary.

The larger lesson is simple: **Herdr can tell a plugin that focus changed, and OMP can pause safely, but the latency between those facts determines the guarantee.**

## Source

- [Herdr — runtime for coding agents](https://github.com/herdrdev/herdr)
- [Oh My Pi — terminal coding agent](https://github.com/can1357/oh-my-pi)
- [Ambient Attention Without Interruption]({% post_url 2026-08-11-ambient-attention-without-interruption %})
