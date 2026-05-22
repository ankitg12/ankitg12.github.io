---
layout: post
title: "No Broken Windows for AI coding agents"
date: 2026-05-22
categories: ai agents productivity
series: "AI coding agent productivity"
---

The broken windows theory says a single unrepaired broken window invites more damage. Left alone, it signals that nobody is maintaining the building.

AI coding agents have broken windows too. They look like this:

```
Error: unknown flag: --limit
```

The agent notes it, moves on, and tries something else. The broken window stays broken. Next session, same agent, same wrong flag — it hits it again. The skill file that generated `--limit` is never fixed. The CLI changed six months ago. Nobody updated the prompt.

This is not a model failure. It is a maintenance failure.

## The pattern

Every tool call failure that isn't a network error or user abort falls into one of two categories:

1. **Agent mistake** — bad anchor, wrong parameter shape, logic error. These are ephemeral; no source to patch.
2. **Stale env knowledge** — wrong CLI flag, dead path, renamed command, changed API. These have a *source*: a skill file, a command template, a CLAUDE.md entry, a notes file. The source is wrong and will cause the same failure again.

Category 2 is the broken window. The agent sees it, works around it, and moves on. The source is never updated. The window stays broken.

## The fix: a post-tool-call hook

The idea is simple: when a tool call fails, spawn an autonomous sub-task that finds and patches the source before continuing.

```
tool fails
  → boomerang sub-task fires
  → searches skills, commands, notes for the source of the call
  → patches it
  → context collapses to a summary
  → main session continues
```

The main agent never retries the broken call. It gets a summary: "Found `--limit` in `daily-jira-check.md`, patched to `--paginate`." The window is repaired.

## Implementation: OMP extension

OMP (Oh My Pi) exposes a `tool_result` event that fires after every tool execution. When `isError` is true, the extension fires.

```typescript
import type { ExtensionAPI } from "@oh-my-pi/pi-coding-agent";

export default function ompNbw(pi: ExtensionAPI) {
  pi.on("tool_result", async (event: any, ctx) => {
    if (!event.isError) return;

    const errorLine = extractFirstErrorLine(event.content);

    pi.sendUserMessage(
      `/boomerang NBW: \`${event.toolName}\` failed with "${errorLine}". ` +
      `Search skills, commands, notes, and config files for the source. ` +
      `Patch it. No need to report back — just fix it.`,
      { deliverAs: "followUp" },
    );
  });
}
```

`deliverAs: "followUp"` means the boomerang task fires immediately after the current agent turn ends. The `/boomerang` prefix hands it to [pi-boomerang](https://github.com/nicobailon/pi-boomerang), which runs the sub-task autonomously and collapses the context to a summary when done.

The full extension with status bar feedback, session count, and `/nbw-off`/`/nbw-on` commands is at [ankitg12/omp-nbw](https://github.com/ankitg12/omp-nbw).

## Design decisions

**Universal gate, not pattern matching.** An earlier version only fired on specific error patterns (`unknown flag`, `ENOENT`, `command not found`). That's the wrong instinct — it assumes we know which failures are fixable. The boomerang agent is equally capable of deciding that. Fire on everything; let the agent determine if there's a source to patch.

**Boomerang, not inline fix.** The NBW task runs as a separate boomerang sub-task, not as a continuation of the main agent turn. This keeps the fix work isolated and collapsed. The main session sees a one-line summary, not a wall of detective work. The boomerang agent has the same tools and context — it finds files the same way the main agent would.

**`followUp`, not `nextTurn`.** `followUp` fires after the current turn ends without user action. `nextTurn` parks until the user submits input. For NBW, the fix should be automatic — the user opted into self-healing by installing the extension. If you want human review before the fix fires, switch to `nextTurn` and treat the queued boomerang prompt as a proposal.

**Status bar as signal.** The OMP status bar shows `🪟 NBW active` when the extension is running and `🪟 NBW #N — <tool> failed` when a fix is in flight. Resets to active on `agent_end`.

## What it actually found

First real test: a notes file called `daily-jira-check.md` containing:

```bash
jira issue list -s "In Progress" --count 5 --plain
```

`--count` is not a valid flag. The correct flag is `--paginate`. The boomerang agent:

1. Received the error: `unknown flag: --count`
2. Searched `notes/`, `skills/`, `commands/` for `--count`
3. Found `daily-jira-check.md` as the most recently modified file in `notes/`
4. Patched line 6: `--count 5` → `--paginate 5`
5. Collapsed to a summary

The main session saw: `[BOOMERANG COMPLETE] Patched --count to --paginate in daily-jira-check.md`.

The window was repaired. Next run of that command will work.

## Limitations

- The boomerang agent searches files it can find. If the source is in a binary, a compiled prompt, or a remote config, it won't find it.
- Truly ephemeral failures (network timeout, rate limit) will still fire NBW unnecessarily. The agent recognizes these and does nothing — but it costs a boomerang sub-task.
- Context collapse means the fix detail is summarized, not preserved verbatim. If you want a full audit trail, add logging in the extension before the boomerang fires.

## The bigger point

Every time an agent hits a failure and works around it, it is choosing not to fix the building. The broken windows accumulate. The agent's environment degrades slowly, invisibly, until sessions are full of workarounds for things that should have been fixed at the source.

NBW is one line of policy: failures are not to be tolerated, they are to be repaired.

---

**Source:** [ankitg12/omp-nbw](https://github.com/ankitg12/omp-nbw)  
**Requires:** [pi-boomerang](https://github.com/nicobailon/pi-boomerang), OMP (Oh My Pi)
