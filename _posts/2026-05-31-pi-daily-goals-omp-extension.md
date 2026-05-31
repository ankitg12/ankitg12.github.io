---
layout: post
title: "Building a daily goals widget for OMP — a full extension walkthrough"
date: 2026-05-31
categories: ai agents productivity tools
series: "AI coding agent productivity"
---

The editor is open. The context window is filling up. Somewhere in your `daily-goal.md` are the seven things you committed to getting done today. None of them are visible without switching to another terminal. You forget what you were doing. The agent keeps working. You've drifted.

This is the story of building `pi-daily-goals` — an OMP extension that pins today's goals below the editor, with collapsible states, an active-goal highlight, and a status bar feed. It took two iterations, surfaced a runtime crash, uncovered an architectural mistake, required reading OMP source code to understand what's actually possible, and ended with the extension tracking itself as goal #8 of the day it was built.

---

## What it does

A `daily-goal.md` file at `~/.omp/agent/daily-goal.md` holds today's date and goals in plain YAML:

```yaml
date: 2026-05-31
active: 3
goals:
- Security Awareness Training (TalentConnect) — P0 overdue Dec 2025
- POSH at Workplace India (TalentConnect) — P0 overdue Apr 2026
- Fix Jira CLI auth — jira token set
- Fix web_search — jina.ai free tier
- Retro/diary catch-up — missing journal entries
- LinkedIn profile — apply draft
- Briefing context — prune stale threads
```

On every session start (and at turn end), the extension reads this file, checks the date, and renders a widget below the OMP editor:

```
 Daily Goals ▾ — 2026-05-31
  ○ 1. Security Awareness Training (TalentConnect) — P0 overdue Dec 2025
  ○ 2. POSH at Workplace India (TalentConnect) — P0 overdue Apr 2026
  ◉ 3. Fix Jira CLI auth — jira token set          ← highlighted in warning colour
  ○ 4. Fix web_search — jina.ai free tier
  ...
```

The status bar simultaneously shows `◉ Fix Jira CLI auth — jira token set (3/7)`.

Three commands:

| Command | Effect |
|---|---|
| `/showgoals` | Cycle: expanded → collapsed → hidden → expanded |
| `/showgoals N` | Mark goal N as active; writes `active: N` back to the file |

If the date in the file doesn't match today, the widget is silent. No stale goals, no noise.

---

## The extension file structure

OMP extensions are directories with a `package.json` pointing to a `.ts` entry point. No build step — OMP compiles TypeScript directly at load time via Bun.

```
~/repos/github.com/ankitg12/pi-daily-goals/
├── pi-daily-goals.ts
└── package.json
```

```json
{
  "name": "@ankitg12/pi-daily-goals",
  "pi": {
    "extensions": ["./pi-daily-goals.ts"]
  }
}
```

Register in `~/.omp/agent/config.yml`:

```yaml
extensions:
  - ~/repos/github.com/ankitg12/pi-daily-goals
```

OMP expands `~` in both paths. The directory path is registered, not the `.ts` file — OMP reads `package.json` to find the entry point.

**Development workflow:** start at `~/.omp/agent/extensions/<ext-name>/` (no git, fast iteration, identical structure), then `mv` to `~/repos/github.com/<user>/<ext-name>/` when working and update the config entry.

---

## First version — working but wrong in two ways

The first version worked immediately. Goals showed up above the editor on session start. Then two bugs surfaced.

### Bug 1: `theme.fg("fg")` crashes

The widget factory called `theme.fg("fg", goalText)` to apply standard foreground colour:

```typescript
lines.push(`  ${theme.fg("accent", `${i + 1}.`)} ${theme.fg("fg", goals[i])}`);
```

OMP threw `Unknown theme color: fg` at load time. The extension silently died — no goals, no error message in the UI beyond the startup warning.

`"fg"` is not a valid theme colour key. The valid keys are `"accent"`, `"muted"`, `"dim"`, `"warning"`, `"text"`. For terminal-default foreground — the most common case — just use a bare string. The terminal inherits the foreground colour without any escape sequence:

```typescript
lines.push(`  ${theme.fg("accent", `${i + 1}.`)} ${goals[i]}`);  // bare string works
```

### Bug 2: re-registering the factory on every turn

The first version called `ctx.ui.setWidget(...)` on every `turn_end`, every `session_start`, every `session_switch`:

```typescript
pi.on("turn_end", (_event, ctx) => {
  // ❌ This re-registers the factory on every turn
  ctx.ui.setWidget(WIDGET_ID, (_tui, theme) => {
    return { render: () => buildLines(), invalidate: () => {} };
  });
});
```

This is wrong. `setWidget` accepts a factory that OMP calls **once** to mount a component. The component's `render(width)` is called each frame. Re-registering on every turn replaces the mounted component, which works but causes unnecessary remounting and doesn't take advantage of the differential renderer.

The correct pattern — confirmed from reading `rpiv-todo`, the canonical OMP widget extension — is register-once and use `tui.requestRender()` for all subsequent redraws.

---

## What OMP actually is

Before the rewrite, I assumed OMP's TUI was built on [blessed](https://github.com/chjj/blessed) — reasonable guess, since it's the dominant Node.js terminal library and blessed supports draggable boxes. Wrong assumption.

OMP (`oh-my-pi`) uses a completely custom differential terminal renderer in `packages/tui` (~66KB of TypeScript). Dependencies: `@oh-my-pi/pi-natives`, `@oh-my-pi/pi-utils`, `lru-cache`, `marked`. No blessed, no Ink, no React. It renders only changed cells, similar to ncurses but implemented from scratch in Bun-native TypeScript.

This matters for extension development:

- No blessed widget API → no draggable boxes, no layout tree, no z-indexed floats
- No DOM → no CSS-style positioning
- The widget API (`setWidget`) produces lines of text that OMP places in a fixed panel; the extension author has no control over position beyond `"aboveEditor"` / `"belowEditor"`
- `ctx.ui.custom(..., { overlay: true })` exists for modal overlays but they're blocking — they take keyboard focus until `done()` is called

The source is in ghq if you have it cloned: `~/repos/github.com/can1357/oh-my-pi`. Reading it directly answered questions that no amount of searching could.

---

## Reading the official docs

OMP has full documentation in `packages/coding-agent/docs/`. Key files:

- `extensions.md` — full API surface, all hook events, ctx fields
- `tui.md` — component contract, rendering constraints, overlay API
- `theme.md` — valid colour keys, syntax highlighting tokens

And `pi.dev/packages` lists the ecosystem of published extensions. Two are relevant here:

| Package | What it does |
|---|---|
| `@juicesharp/rpiv-todo` | Todo list as a live overlay — canonical setWidget reference |
| `pi-powerline-footer` | Powerline-style bar; reads `setStatus` from other extensions via `customItems` |

`rpiv-todo` taught the register-once pattern. `pi-powerline-footer` showed that if your extension publishes to `ctx.ui.setStatus("daily-goals", text)`, powerline users can promote it to their bar with zero extra work.

---

## The correct widget architecture

The Component interface from `tui.md`:

```typescript
interface Component {
  render(width: number): string[];
  handleInput?(data: string): void;
  wantsKeyRelease?: boolean;
  invalidate(): void;
}
```

`render(width)` is called every frame. `invalidate()` is called when OMP resets the widget slot (e.g. after `/reload`). Extensions import `truncateToWidth` and `replaceTabs` from `@oh-my-pi/pi-tui` to produce safe output:

```typescript
import { replaceTabs, truncateToWidth } from "@oh-my-pi/pi-tui";

render(width: number): string[] {
  return buildLines(state, mode, theme)
    .map(l => truncateToWidth(replaceTabs(l), width));
}
```

The register-once pattern:

```typescript
let tui: any = null, theme: any = null, widgetRegistered = false;

const widget = {
  render(width: number): string[] { /* ... */ },
  invalidate(): void {
    tui = null;
    widgetRegistered = false;  // force re-registration on next refresh
  },
};

function refresh(ctx: any): void {
  if (!widgetRegistered) {
    widgetRegistered = true;
    ctx.ui.setWidget(WIDGET_ID, (t: any, th: any) => {
      tui = t; theme = th;
      return widget;  // return the Component, not { render, invalidate }
    }, { placement: "belowEditor" });
  } else {
    tui?.requestRender();  // triggers render(width) without re-mounting
  }
}
```

To remove the widget:

```typescript
ctx.ui.setWidget(WIDGET_ID, undefined);
widgetRegistered = false; tui = null;
```

---

## `belowEditor` placement

`setWidget` accepts an optional third argument:

```typescript
type WidgetPlacement = "aboveEditor" | "belowEditor";
ctx.ui.setWidget(id, factory, { placement: "belowEditor" });
```

`"belowEditor"` places the widget between the editor input and the status bar — exactly where a persistent context strip belongs. The default (`"aboveEditor"`) places it between the chat history and the editor. Neither option is below the status line: that area isn't exposed to extensions.

`setFooter` and `setHeader` exist in the type definitions but are documented as **no-ops in interactive mode**. Don't use them.

---

## Active goal tracking

The `daily-goal.md` format is extended with an optional `active: N` field (1-indexed):

```yaml
date: 2026-05-31
active: 3
goals:
- ...
```

Parsing:

```typescript
const activeMatch = raw.match(/^active:\s*(\d+)/m);
const activeN     = activeMatch ? parseInt(activeMatch[1], 10) : 0;
const activeIdx   = activeN >= 1 && activeN <= goals.length ? activeN - 1 : -1;
```

`/showgoals 3` writes the field back:

```typescript
function writeActive(n: number): void {
  let raw = readFileSync(GOAL_FILE, "utf-8");
  if (/^active:\s*\d+/m.test(raw)) {
    raw = raw.replace(/^active:\s*\d+/m, `active: ${n}`);
  } else {
    raw = raw.replace(/^(date:\s*\S+)/m, `$1\nactive: ${n}`);
  }
  writeFileSync(GOAL_FILE, raw, "utf-8");
}
```

The active goal renders in `theme.fg("warning", text)` — the amber highlight colour — with a `◉` icon instead of `○`. In collapsed mode, only the active goal line is shown:

```
 Daily Goals ▸ — 2026-05-31
  3. Fix Jira CLI auth — jira token set (3/7)
```

---

## Three-state collapsible

Widget mode is a three-value cycle managed in extension state:

```typescript
type WidgetMode = "expanded" | "collapsed" | "hidden";
```

`/showgoals` advances the cycle. The arrow in the header indicates current state (`▾` expanded, `▸` collapsed). Hidden removes the widget entirely but keeps the status bar entry alive — so even when the widget is off, `◉ Fix Jira CLI (3/7)` remains visible in the status line.

---

## Command naming: `/showgoals` not `/goals`

The first version registered `/goals`. OMP has a built-in `/goal` command. Close enough to cause confusion — muscle memory for one fires the other. Renamed to `/showgoals`, which is unambiguous and self-documenting.

---

## Status bar integration

Every refresh publishes to the status bar:

```typescript
ctx.ui.setStatus("daily-goals",
  activeIdx >= 0
    ? `◉ ${short(goals[activeIdx])} (${activeIdx + 1}/${goals.length})`
    : `Goals: ${goals.length} today`
);
```

The key `"daily-goals"` is stable. `pi-powerline-footer` users can pull it into their bar by adding to their config:

```json
{
  "customItems": [{ "statusKey": "daily-goals", "position": "secondary" }]
}
```

---

## What didn't work

**Draggable widget.** Blessed supports `draggable: true` on any box. OMP's TUI doesn't have this concept — the widget API produces lines of text, not positioned elements. Not possible without forking OMP.

**Non-blocking floating overlay.** `ctx.ui.custom(..., { overlay: true })` mounts a bottom-centered modal overlay — it takes keyboard focus and blocks the editor until `done()` is called. There's no always-on-top non-modal float. The belowEditor widget is the correct ambient pattern.

**`setFooter` / `setHeader`.** Both are no-ops in interactive mode per the docs. The powerline footer extension works by `setWidget(..., { placement: "belowEditor" })` combined with its own internal layout, not by these methods.

**Re-reading the file on every frame.** `render(width)` is called every rendered frame. File I/O inside `render()` would be catastrophic. The YAML parsing happens only on `session_start`, `session_switch`, `turn_end`, and command invocation — then the result is held in extension state that `render()` reads from.

---

## The self-referential close

The extension was shipped before it appeared in `daily-goal.md` — a goals tracker that wasn't tracking its own goal. Added retroactively as goal #8:

```yaml
- Build pi-daily-goals OMP extension — belowEditor widget, /showgoals, active goal highlight ✓
```

Then `/showgoals 8`. The status bar showed:

```
◉ Build pi-daily-goals OMP extension —… (8/8)
```

The extension tracking itself as the active goal on the day it was built. The loop closed.

---

## Source

- [pi-daily-goals](https://github.com/ankitg12/pi-daily-goals) — the extension (two files, ~180 lines of TypeScript)
- [rpiv-todo](https://github.com/juicesharp/rpiv-mono/tree/main/packages/rpiv-todo) — canonical `setWidget` + register-once pattern reference
- [oh-my-pi](https://github.com/can1357/oh-my-pi) — OMP source; `docs/tui.md`, `docs/extensions.md`, `docs/theme.md` are the authoritative API reference
- [pi.dev/packages](https://pi.dev/packages) — extension ecosystem; browse before building
- [AFK toggle]({% post_url 2026-05-27-afk-toggle-for-ai-coding-agents %}) — `ctx.ui.setStatus` pattern
- [Session goal tracking]({% post_url 2026-05-11-session-goal-tracking-for-ai-agents %}) — the daily-goal.md format this extension reads
