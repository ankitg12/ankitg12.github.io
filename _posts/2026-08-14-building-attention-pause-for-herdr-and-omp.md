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

The first version therefore did not invent another pause mechanism. It submitted the command OMP already understood. A live drafting failure the next morning proved that the transport was wrong; the [August 15 update](#update-the-prompt-editor-is-not-a-control-plane) below replaces it with an out-of-band key action while retaining OMP's process-wide pause gate.

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
def omp_candidates(agents, *, include_focused=False):
    return [
        agent
        for agent in agents
        if agent.get("agent") == "omp"
        and agent.get("agent_status") in {"idle", "working", "done"}
        and (include_focused or not agent.get("focused", False))
    ]
```

A pane can contain old `P A U S E D` text in its scrollback, so detecting that heading alone is unsafe. The handler also requires the live pause footer:

```python
PAUSE_FOOTER = "esc · enter · space — resume"


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

## Scheduled breaks need ownership, not polling

Focus changes are not the only reason to pause every agent. Windows Task Scheduler
already owns my Stretchly schedule, including a 35-second mini break every ten
minutes and a longer break at `:50`. Making every OMP session watch Stretchly would
duplicate state, create one poller per process, and make resume decisions locally.

Instead, the Scheduler invokes the same Herdr controller directly:

```text
attention_pause.py break-start --run-id <guid>
Stretchly renders the break
attention_pause.py break-end --run-id <guid>
```

`break-start` captures Herdr's focused pane and requests OMP's native pause for every
eligible OMP pane, including the focused one. The first implementation submitted
`/pause` as a prompt; the August 15 repair routes the request through a dedicated
control key instead. While a break is active, normal `pane.focused` reconciliation
cannot exempt an agent. Break end clears the hold but resumes nothing; I choose
which agent deserves attention next.

The controller stores a set of active run IDs rather than one Boolean. That matters
because the `:50` long-break process can still be sleeping when the `:00` task
starts. An older invocation may remove only its own token, so it cannot falsely end
the newer break. An optional flag resumes only the pane captured as last active,
never every OMP process.

The 20:20 live test produced three independent pieces of evidence: the Scheduler
logged its unique run ID, the controller's ownership set was empty after the break,
and Herdr detection screens showed two agents still paused while only the pane I
selected had resumed. The complete plugin test file also passed all 17 tests,
including overlap and default no-resume cases.

This path removes event-discovery ambiguity because the component that owns the
schedule initiates the policy directly. It does not make enforcement real-time:
in-flight calls still finish, and OMP pauses only at its next safe boundary.

## Update: the prompt editor is not a control plane

On August 15, a real interaction exposed a more serious defect than latency. I was
writing a prompt when focus changed. The plugin ran this line:

```python
herdr_json(herdr, "agent", "prompt", pane_id, "/pause")
```

Herdr's `agent prompt` operation submits text through OMP's editor. The editor was
already holding my draft, so `/pause` joined user text instead of remaining a local
command. The plugin had treated a shared data channel as a control channel.

Clearing the editor first would merely exchange prompt corruption for prompt loss.
Capturing, clearing, submitting, and restoring the draft would still race with the
user's keyboard. The repair therefore had one non-negotiable property: **pause must
not write text into the editor at all**.

OMP's RPC mode was not an escape hatch. `omp --mode rpc` creates a separate process
that reads JSON commands from its own standard input; it does not attach to an
already-running interactive TUI, and its command union has no process-global pause
operation.

OMP extensions already provide the needed control surface: a keyboard shortcut can
run code without entering editor text. A small extension binds `Ctrl+Alt+P` directly
to OMP's existing process-wide `agentPauseGate`. The Python controller now sends
that logical key through Herdr:

```python
PAUSE_SHORTCUT = "ctrl+alt+p"


def request_pause(herdr, pane_id):
    herdr_json(herdr, "agent", "send-keys", pane_id, PAUSE_SHORTCUT)
```

The extension engages the same pause gate used by native `/pause`, presents a pause
overlay, and resumes on Escape, Enter, Space, or `Ctrl-C`. Main-agent, subagent, and
advisor semantics therefore remain process-wide; only the transport changed.

The live regression test used a fresh OMP process in a disposable Herdr pane:

1. Type `DRAFT-SENTINEL-7f31 keep this exact` without pressing Enter.
2. Send `Ctrl+Alt+P` through `herdr agent send-keys`.
3. Confirm that the pause overlay appears.
4. Resume with Space.
5. Read the editor again and compare the draft byte for byte.

The draft remained exact, and no `/pause` text appeared in it.

This repair also found a detection edge case. OMP's wide pause screen ends with
`esc · enter · space — resume`, while a narrow or short pane ends with
`esc to resume`. Detection now requires the common `P A U S E D` title within the
final five non-empty lines *and* one of those exact final footers. A realistic wide
fixture matters: its two body lines and clock place the title fifth from last, so a
four-line tail check silently misses the native screen.

The resulting test suite passes 18 tests, including compact and full native layouts,
stale scrollback text, focus races, overlapping scheduled breaks, and the out-of-band
`send-keys` request. The current plugin subscribes only to `pane.focused`; removing
`pane.agent_status_changed` avoids a feedback loop in which pause-induced state
changes start more reconciliation handlers.

The revised lesson is stronger than "use the native pause mechanism": **use the
native state transition through a control channel that cannot collide with user
data.**

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
2. Leave an identifiable unsent draft in agent A.
3. Switch from agent A to agent B.
4. Record the `pane.focused` event time.
5. Confirm A receives the out-of-band control key and shows its pause overlay.
6. Return to A, resume it manually, and confirm the draft is byte-identical.
7. Repeat rapidly enough to overlap handlers.
8. Repeat while one OMP tool call is in flight.
9. Correlate focus, key dispatch, overlay, and session-log timestamps.
10. Reject a deterministic claim if either agent starts new work after its safe boundary.

Unit tests prove selection rules. Only this end-to-end test proves that Herdr events, subprocess startup, CLI transport, terminal state detection, and OMP control compose into the intended attention boundary.

The larger lesson is simple: **Herdr can tell a plugin that focus changed, and OMP can pause safely, but the latency between those facts determines the guarantee.**

## Source

- [Herdr — runtime for coding agents](https://github.com/herdrdev/herdr)
- [Oh My Pi — terminal coding agent](https://github.com/can1357/oh-my-pi)
- [Ambient Attention Without Interruption]({% post_url 2026-08-11-ambient-attention-without-interruption %})
