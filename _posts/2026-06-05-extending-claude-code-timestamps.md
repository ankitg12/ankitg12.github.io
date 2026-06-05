---
layout: post
title: "Extending claude-code-timestamps: three things that didn't work and what to do instead"
date: 2026-06-05
categories: ai agents productivity
series: "AI coding agent productivity"
---

[claude-code-timestamps](https://github.com/ankitg12/claude-code-timestamps) shipped in late May: 12 CC lifecycle events, one Python script, zero dependencies. I've been running it daily since. When CC 2.1.163 dropped with three hook enhancements, I spent a session trying to exploit them. All three hit walls. This post documents what failed and what the right approach turned out to be.

---

## What v2.1.163 added for hooks

Three relevant changes in the release:

**Stop and SubagentStop hooks can now return `additionalContext`.** Previously this output type was model-facing only and wouldn't stop the session from ending. The release note said these hooks can now "continue the turn without being labeled a hook error."

**`MessageDisplay` hook** (added in v2.1.152, now stable) — fires as assistant messages stream to the terminal. Can return `displayContent` to replace what the user sees on screen.

**`UserPromptSubmit` hook `additionalContext`** — already existed; confirmed stable. Injects text into Claude's context without the user seeing it.

---

## Attempt 1: additionalContext on Stop

The plan: emit timing data (turn duration, session total) into Claude's context via `additionalContext` on the `Stop` hook. Then Claude can reference exact timing in retros and answers without needing to look it up.

The release note said "continue the turn." I read "continue" as "proceed normally." It means "trigger another Claude response."

**What happened:** After every turn, Claude emitted a response to its own timing data. "Turn took 54.43s. Session total: 11m28s." → Claude: *"The user hasn't responded yet — this is just a stop hook firing. I should wait for their response."*

The fix: revert Stop to plain `systemMessage`. `additionalContext` on Stop is only useful when you want Claude to keep working after the turn ends — not for passive timing annotations.

**Rule:** `additionalContext` on Stop = "Claude, do more work." Use `systemMessage` if you just want a visible banner.

---

## Attempt 2: MessageDisplay for response timestamps

The plan: prepend `[HH:MM:SS AM]` to each assistant response as it streams, so every Claude message is visibly timestamped. You can then say "the message at 12:32" and I have that anchor.

Three things blocked this:

**Plugin dispatch bug (Issue #10875).** Plugin hooks — defined in `plugin.json` — execute successfully but their JSON stdout is not captured by CC. The hook runs, the script exits 0, the output is discarded. This affects all plugin-defined hooks. Hooks only work reliably when defined directly in `settings.json`.

The fix: move all hook definitions from `plugin.json` to `~/.claude/settings.json`, pointing at the plugin cache path directly. The plugin becomes a distribution vehicle only; the actual hook wiring lives in settings.

**`--verbose` bypasses `displayContent`.** From the docs: "verbose mode shows the original." If you launch with `claude --verbose` (or have `"verbose": true` in settings), `MessageDisplay` hooks' `displayContent` replacements are ignored entirely. The terminal shows the unmodified text regardless of what `displayContent` returns.

**CC doesn't fire `MessageDisplay` from chunk index 0.** Debug logging showed the hook firing from index 3, 5, sometimes higher — never 0. The first chunk of each message is displayed before the hook runs. A sentinel-file approach (emit the timestamp on whichever chunk comes first) partially works, but the timestamp appears mid-message after streaming has already started.

Combined: `MessageDisplay` is unreliable for this use case if you use verbose mode at all, and even without it the first-chunk timing means you see the timestamp on a partially-rendered message.

---

## What actually works: additionalContext on UserPromptSubmit

The real goal was: you say "the message at 12:32" and I have the timestamp in my context to reference it. `MessageDisplay` was the wrong hook for this.

`UserPromptSubmit` hooks can emit both `systemMessage` (visible banner) and `additionalContext` (injected into Claude's reasoning context) in a single response:

```python
print(json.dumps({
    "systemMessage": f"[{ts}] — user message received{idle_part}",
    "hookSpecificOutput": {
        "hookEventName": "UserPromptSubmit",
        "additionalContext": f"User message received at {ts}.{idle_part}",
    }
}))
```

The `systemMessage` shows the banner in the transcript. The `additionalContext` appears in the system-reminder block at the top of my context window:

```
UserPromptSubmit hook additional context: User message received at 12:42:12 PM.
```

Now when you reference a message time I can confirm it, find it in the conversation, and reason about elapsed time without any ambiguity.

---

## The plugin dispatch bug in detail

The symptom: `PostToolUse` hook was configured as `async: true` in `plugin.json`. CC showed "Async hook PostToolUse completed" in the transcript — the hook ran — but no timestamp banner appeared. Switching to sync didn't help. Identical configuration in `settings.json` worked immediately.

The issue is in how CC invokes plugin-scoped hooks vs settings-scoped hooks. Plugin hooks skip the stdout capture step. The [GitHub issue #10875](https://github.com/anthropics/claude-code/issues/10875) has the details: "Plugin hooks execute successfully, but their JSON output is not captured/parsed by Claude Code. Identical hook files work correctly as inline hooks in settings.json."

This affects every plugin that relies on hook output — `systemMessage`, `additionalContext`, `displayContent`, `decision: block`, all of it.

**Workaround:** Define hooks in `settings.json`. Point at the plugin cache path:

```json
"PostToolUse": [{
  "matcher": "",
  "hooks": [{
    "type": "command",
    "command": "python C:/Users/you/.claude/plugins/cache/ankitg12/claude-code-timestamps/1.0.0/timestamps.py tool"
  }]
}]
```

The plugin cache path is stable across updates as long as the version doesn't change. When you update the plugin, update the path in settings too.

---

## What changed in v1.2.x

**v1.1.0** — added `combined_output()` helper to emit both `systemMessage` and `additionalContext` in one hook response. Used on SubagentStop (which doesn't continue the turn).

**v1.2.0** — added `MessageDisplay` mode (prepend timestamp to first chunk). Works without `--verbose`. Documented the verbose incompatibility.

**v1.2.1** — `UserPromptSubmit` now emits `additionalContext` alongside the visible banner. Plugin disabled in favour of settings.json wiring. Stop hook reverted to plain `systemMessage` after the additionalContext-continues-turn discovery.

---

## Current state

What works reliably, wired via `settings.json`:

```
[12:39:46 AM] — user message received          ← UserPromptSubmit (visible + in Claude's context)
[12:39:48 AM] Read completed                   ← PostToolUse
[12:39:52 AM] Glob completed                   ← PostToolUse
[12:40:10 AM] — response complete (24.3s) | session 8m12s   ← Stop
```

What doesn't work (and why):
- `additionalContext` on Stop → triggers another Claude response
- `MessageDisplay` with `--verbose` → displayContent silently ignored
- Plugin-defined hooks → stdout not captured (Issue #10875)

---

## Source

- [claude-code-timestamps v1.2.1](https://github.com/ankitg12/claude-code-timestamps)
- [Claude Code timestamps: 12 lifecycle events, one script]({% post_url 2026-05-29-claude-code-timestamps %}) — the original post
- [CC hooks reference](https://code.claude.com/docs/en/hooks) — full event and output schema
- [Issue #10875 — Plugin hooks JSON output not captured](https://github.com/anthropics/claude-code/issues/10875)
- [Issue #44763 — Show timestamps on conversation messages](https://github.com/anthropics/claude-code/issues/44763)
