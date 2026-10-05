---
layout: post
title: "Decaying Effort: A Transmission Gearbox for AI Coding Agents"
date: 2026-10-05 22:05:00 +0530
categories: ai agents productivity
series: "AI coding agent productivity"
---

When an AI coding session begins, the problem space is at its widest. The architecture is undefined, constraints are unverified, and a subtle misunderstanding early on will derail dozens of downstream turns. Running at maximum reasoning effort (`max` or `xhigh`) during this opening phase pays for itself.

Ten turns later, the mode changes. The plan is agreed upon, interfaces are settled, and the agent is writing unit tests, adding fields to a schema, or running linter cleanups. Continuing to run every follow-up turn at maximum thinking effort burns tokens, slows response velocity, and adds latency without improving precision.

Manually toggling `/effort` every few prompts is friction that developers inevitably abandon.

The solution is a **transmission gearbox schedule**: start in first gear with maximum reasoning effort, downshift sequentially across conversational turns, and cruise in `auto` where the model dynamically sizes its own thinking.

---

## The Gearbox Schedule

Just like an automobile transmission, high torque is required to overcome inertia from a dead stop, while cruising on a highway requires far less rotational resistance.

| Gear | Stage | Effort Level | Primary Objective |
| :--- | :--- | :--- | :--- |
| **1st Gear** | Turn 1 | `max` | Architectural grounding, root-cause investigation, tradeoff analysis |
| **2nd Gear** | Turn 2 | `xhigh` | Detailed design, interface contracts, error strategy |
| **3rd Gear** | Turn 3 | `high` | Core implementation and logic structure |
| **4th Gear** | Turn 4 | `medium` | Test scaffolding, edge case coverage, refactoring |
| **Cruising** | Turn 5+ | `auto` | Incremental iterations, diagnostics, mechanical edits |

If the active model only supports a subset of efforts (such as models that support `[low, medium, high]`), the requested level clamps cleanly to the model's highest supported tier without breaking the sequence.

---

## Prior Art & Ecosystem Search (DRW)

Before building this extension, we surveyed the public Pi / Oh-My-Pi (OMP) ecosystem and GitHub repositories:

| Extension / Repo | Mechanism | Decaying Schedule? |
| :--- | :--- | :--- |
| `ricardofrantz/pi-effort` | Interactive slash command to manually pick effort | ❌ No |
| `robzolkos/pi-skill-model-effort` | Frontmatter metadata mapping effort to active skill | ❌ No |
| `nicobailon/pi-boomerang` | Snapshots and switches effort inside isolated sub-task runs | ❌ No |

No existing tool automated a decaying effort schedule across conversational turns.

---

## Implementation Details

The extension `@ankitg12/decaying-effort-omp` hooks into OMP's extensibility API.

### 1. Multi-Model Support

The extension uses `pi.setThinkingLevel()`, which maps natively across providers:
- **Anthropic**: Translates to thinking budget and adaptive modes.
- **OpenAI**: Maps to `reasoning: { effort: "high" | "medium" | "low" }`.
- **Gemini**: Maps to native thinking levels.

### 2. Manual User Override Discipline

If a developer manually overrides the thinking level mid-session using `/effort` or the UI menu, the extension detects the divergence, logs `user_override_detected`, and halts automated downshifts. The developer's explicit intent is never clobbered.

### 3. Transcript-Aware Resumption

A naive turn counter resets on session reload or branch switch, which would mistakenly drop an 80-turn conversation back into 1st gear (`max`).

On `session_start` and `session_switch`, the extension inspects `ctx.sessionManager.getEntries()`, counts prior user turns, and lands immediately in the appropriate cruising gear:

```typescript
const resetOrResumeState = (reason: string, ctx?: any) => {
	const entries = ctx?.sessionManager?.getEntries?.();
	if (Array.isArray(entries) && entries.length > 0) {
		const userTurns = entries.filter(
			(e: any) => e.type === "message" && e.message?.role === "user"
		).length;
		turnCount = userTurns;
		logJSONL("session_resumed", { reason, turnCount, historyEntries: entries.length });
	} else {
		turnCount = 0;
		logJSONL("session_started", { reason });
	}
	userOverridden = false;
	lastProgrammaticLevel = undefined;
};
```

### 4. Structured JSONL Observability

Rather than relying solely on transient terminal notifications, the extension writes structured records to `~/.omp/logs/decaying-effort.jsonl`:

```json
{"ts":"2026-10-05T16:29:54.532Z","event":"session_resumed","turn":22,"currentLevel":"high","userOverridden":false,"reason":"start","turnCount":22,"historyEntries":729}
{"ts":"2026-10-05T16:32:16.050Z","event":"effort_stepped","turn":23,"currentLevel":"high","userOverridden":false,"turnCount":23,"targetLevel":"auto","effective":"high","clamped":false}
```

---

## Interactive Controls

Developers can inspect or adjust the schedule on the fly:

```text
/effort-decay                               # View current turn, level, and schedule
/effort-decay off                           # Pause decay for the current session
/effort-decay on                            # Re-enable decay
/effort-decay reset                         # Rewind counter to Turn 1 (max)
/effort-decay schedule max,high,medium,auto # Set a custom decay curve
```

---

## Source

- **Extension repository:** [ankitg12/omp-extensions](https://github.com/ankitg12/omp-extensions/tree/main/packages/decaying-effort-omp) (`packages/decaying-effort-omp`)
- **OMP Core Extensibility:** [can1357/oh-my-pi](https://github.com/can1357/oh-my-pi)
