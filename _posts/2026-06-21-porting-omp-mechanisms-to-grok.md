---
layout: post
title: "Porting OMP productivity mechanisms to Grok — AFK, suggester, retro"
date: 2026-06-21
categories: ai agents productivity
series: "AI coding agent productivity"
---

OMP extensions get `ExtensionAPI`: `sendUserMessage({ deliverAs: "steer" })`, status bar labels, `setTimeout` timers inside the session process. Grok gets lifecycle hooks, skills, and `/loop` — no steer API, passive hook stdout often ignored. Four mechanisms were ported from OMP/Claude Code to Grok on the personal Mac (2026-06-21). Timestamps failed; three others shipped with degraded but usable parity.

---

## What OMP has that Grok lacks

| OMP capability | Grok equivalent | Gap |
|---|---|---|
| `deliverAs: "steer"` | — | No mid-turn user-message injection |
| `deliverAs: "followUp"` | Stop `additionalContext`, `/loop` | Weaker, turn-boundary only |
| `setTimeout` in-process | Hooks + external state files | No in-harness timer |
| `ctx.ui.setStatusBar` | — | No AFK indicator in TUI |
| `UserPromptSubmit` text transform | — | Cannot prepend `[date time]` to user messages |
| `MessageDisplay` hook | — | Event not supported |

**Implication:** ports are **hook + skill + rules** architectures, not line-for-line extension translations.

---

## Timestamps — the port that didn't work

**Goal:** OMP `chat-timestamps` inline prepend (`[Jun 21, 05:45:53 PM] your message`) plus lifecycle banners from `claude-code-timestamps`.

**What we tried:**
- `timestamps.py` on all CC lifecycle events, wired via `~/.grok/hooks/timestamps.json` + Claude-compat `settings.json`
- `UserPromptSubmit` → `hookSpecificOutput.updatedPrompt` with prepended timestamp
- `PostToolUse` / `Stop` → `systemMessage` banners

**What actually works on Grok:**
- `[ui] show_timestamps = true` in `~/.grok/config.toml` — scrollback times on messages (native, visible)
- Hook scripts run (annotations in hook log) but **stdout is ignored** on passive events per `10-hooks.md`
- `updatedPrompt` on `UserPromptSubmit` does not alter scrollback display in Grok Build/Cursor sessions (verified in `grok-ts-test`)

**Verdict:** Use native `/timestamps` for scrollback. Abandon inline prepend and lifecycle banners on Grok until the harness exposes input-transform or `MessageDisplay`. Keep `timestamps.py` for Claude Code and OMP-compat wiring only.

---

## AFK — works (skills + state + PreToolUse deny)

**OMP:** `agent-afk-omp` — `/afk` sets status bar `AFK 🔴`, `/back` clears; in-memory `afkActive` flag.

**Grok port:**
```
hooks/afk.py                    # state: ~/.claude/afk-state/<session_id>
~/.grok/hooks/afk.json          # PreToolUse deny git push / gh pr when active
skills/afk, skills/back         # engage/disengage via python CLI
rules/afk.md                    # policy when state file exists
```

**Why it works:** AFK only needs (1) durable session state and (2) **blocking** PreToolUse — Grok supports `{"decision":"deny","reason":...}` the same way as Claude Code. No steer injection required.

**Trade-off:** No status-bar indicator; `python3 ~/.claude/hooks/afk.py status` or `/back` exit code is the signal.

---

## Suggester — partial parity (PreToolUse deny for visible nudges)

**OMP:** `agent-suggester-omp` — `setTimeout` + `sendUserMessage({ deliverAs: "steer" })`, editor draft save/restore, `/suggester-{pause,resume,now}`.

**Grok port:**
```
hooks/suggester.py
~/.grok/hooks/suggester.json    # SessionStart, PostToolUse, PreToolUse, SessionEnd
~/.grok/agent/agent-suggester.json
skills/suggester-{pause,resume,now,arm,tick}
rules/suggester.md
```

**Timer:** `PreToolUse` checks elapsed `intervalMs` since `last_fire` — fires on the **next tool call** after the interval, not wall-clock mid-turn like OMP `setTimeout`.

**Injection (v2):** `PreToolUse` **deny** with `🔄 Suggestion` text — same visibility pattern as retro and AFK. Replaced an earlier `Stop` + `additionalContext` path that logged fires but was invisible in Grok Build/Cursor.

**skipIfIdle:** `PostToolUse` increments `tool_call_count`; zero since last fire → skip (same semantics as OMP).

**Optional Grok TUI layer:** `/suggester-arm` → `scheduler_create` or `/loop 10m /suggester-tick` for wall-clock fires during idle gaps.

**Evidence (2026-06-21):**
- `/suggester-now` fired स्थितप्रज्ञ item with exact `wrapWith`; log shows `session_start — armed timer`.
- Timer fire at **20:32:50** via Stop path — logged in `agent-suggester.log` but **not visible** in the session UI.
- Fix in `52ade8e`: moved due nudges to `PreToolUse check block`; verified `decision: deny` + suggestion text blocks the next tool call.

---

## Retro — shared script with Claude Code, weaker Grok delivery

**OMP:** `agent-retro-omp` — clock-aligned `setTimeout`, activity summary, rotate picker across 7 retro lenses.

**Claude Code port:** `retro-tracker.py` — `PreToolUse` **deny** at boundary (blocks next tool until retro runs); `Stop` + `asyncRewake` + exit 2 for session-end retro. See [clock-aligned retro in Claude Code](/ai/agents/productivity/2026/06/21/clock-aligned-retro-claude-code.html).

**Grok port:** same `retro-tracker.py`, `~/.grok/hooks/retro.json`:
```
SessionStart → start
PostToolUse  → tick (async)
PreToolUse   → check block   # deny when boundary passed + activity
Stop         → check         # additionalContext fallback (no asyncRewake)
```

**Grok vs CC on Stop:** CC wakes a new turn via `asyncRewake`. Grok detects `GROK_HOOK_EVENT=stop` and emits `additionalContext` instead — passive, may be low-visibility.

**Enforcement:** `rules/retro.md` — mandatory 6-question response when `Segment: HH:MM →` arrives. Knowledge gem Q5 requires a **live `web_search`**; list every search URL in the retro reply.

**Manual fallback:** `/loop 10m /retro-core`

**Evidence (2026-06-21, Grok Build/Cursor):**
- Armed 20:22:33; boundary **20:30:00**; 10 tool calls in segment.
- Fired **20:31:36** on first `PreToolUse` after boundary (not at :30 exactly — event-driven).
- Log: `fired: 10 calls in segment | mode=check block`
- Deny text: `Segment: 08:22:33 PM → 08:31:36 PM (9min), 10 calls (1 Glob, 1 Grep, 4 Read, 3 Shell, 1 Write)` + Core Retro prompt.
- Hook annotation: `global/settings:pre_tool_use[1].hooks[0] blocked tool Shell`
- Next boundary reset to **20:40:00**.

---

## Parity summary

| Mechanism | OMP | Claude Code | Grok | Injection quality on Grok |
|---|---|---|---|---|
| Timestamps (inline) | extension | hooks | ❌ failed | Native `/timestamps` only |
| Timestamps (banners) | — | hooks | ❌ ignored stdout | — |
| AFK | extension | hooks+skills | hooks+skills | ✅ PreToolUse deny |
| Suggester | extension | — | hooks+skills | ✅ PreToolUse deny |
| Retro | extension | hooks | hooks (shared script) | ✅ PreToolUse deny; Stop weaker |

---

## Portable config: `claude-config/grok/` — not a separate repo

**Question:** Do we need a standalone `grok-config` repo like `claude-config`?

**Answer: No.** Hook scripts, skills, rules, and JSON configs (`agent-suggester.json`, `retro.config.json`) are harness-agnostic and already live in `claude-config`. What Grok adds is thin and machine-specific:

| Artifact | Portable? | Where it lives |
|---|---|---|
| Python hooks (`afk.py`, `suggester.py`, `retro-tracker.py`) | ✅ shared | `claude-config/hooks/` |
| Skills + rules | ✅ shared | `claude-config/skills/`, `rules/` |
| Grok hook JSON (`~/.grok/hooks/*.json`) | ⚠️ paths differ per machine | `claude-config/grok/hooks/*.json.template` |
| `~/.grok/config.toml` | ⚠️ user prefs | `claude-config/grok/config.toml.example` |
| `~/.grok/memory/MEMORY.md` | ❌ personal | stays local, not versioned |
| Claude `settings.json` | ⚠️ Mac vs Windows python | `claude-config/settings.hooks.json` (merge fragment) |

A second repo would duplicate the shared scripts. Instead, `claude-config/grok/` holds templates + `install.sh`:

```bash
git clone git@github.com:ankitg12/claude-config.git ~/ghq/github.com/ankitg12/claude-config
cd ~/ghq/github.com/ankitg12/claude-config
./grok/install.sh          # renders ~/.grok/hooks/*.json, symlinks hooks/skills/config
```

`install.sh` detects `python3`/`python`, substitutes `{{PYTHON}}` and `{{HOOKS_DIR}}` into templates, symlinks `~/.claude/hooks` → repo `hooks/`, and links Grok skills. Override with env vars on Windows:

```bash
PYTHON=python HOOKS_DIR="$HOME/.claude/hooks" ./grok/install.sh
```

Reload: `/hooks` then `r`. Verify: `grok inspect` → Hooks > 0.

**Windows (BLRANGAUR):** merge `claude-config/settings.hooks.json` into `settings.json`; `PYTHON=python`; ghq path may be `~/repos/...` not `~/ghq/...`.

---

## Design rule for future Grok ports

1. **If the feature needs to block an action** → PreToolUse deny (AFK, retro, suggester) — ports cleanly.
2. **If the feature needs to inject a user turn mid-stream** → no clean Grok equivalent; use `/loop`, `scheduler_create`, or accept turn-boundary Stop injection.
3. **If the feature needs to transform display text** → wait for harness API; hooks won't do it today.

Repo: `github.com/ankitg12/claude-config` (hooks, skills, rules, `grok/` templates, `retro.config.json`).

---

## Related posts

- [AFK toggle for AI coding agents](/ai/agents/productivity/2026/05/27/afk-toggle-for-ai-coding-agents.html) — OMP original
- [agent-suggester-omp](/ai/agents/productivity/2026/06/21/agent-suggester-omp.html) — OMP original
- [Clock-aligned retro in Claude Code](/ai/agents/productivity/2026/06/21/clock-aligned-retro-claude-code.html) — CC hooks version
- [claude-code-timestamps](/ai/agents/productivity/2026/05/29/claude-code-timestamps.html) — works on CC, not Grok banners