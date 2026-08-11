---
layout: post
title: "When a New Terminal Tab Is the Wrong Tab"
date: 2026-08-11
categories: wezterm terminal ai agents productivity
series: "AI coding agent productivity"
---

A terminal emulator and a terminal multiplexer can both create tabs, but those tabs do not belong to the same world.

I run [Herdr](https://github.com/herdrdev/herdr), an agent-aware terminal multiplexer, inside [WezTerm](https://wezterm.org/). My WezTerm context menu can send selected text to an AI agent, launch a clean agent, or open a directory in a new tab.

The menu worked until Herdr became the main workspace. Choosing **Ask Jeeves to Debug** still created a tab, but it was a *WezTerm* tab beside Herdr—not a Herdr tab inside the current workspace. Herdr could not track, restore, label, or navigate to it.

The command was correct. The owner of the new tab was wrong.

## Two tab models

The setup has two independent control planes:

```text
WezTerm window
├── WezTerm tab: Herdr
│   ├── Herdr tab: agent A
│   ├── Herdr tab: agent B
│   └── Herdr tab: shell
└── WezTerm tab: accidentally spawned agent   ← invisible to Herdr
```

A native WezTerm action such as `mux_window:spawn_tab()` always adds a WezTerm tab. It cannot infer that the foreground program is itself a multiplexer with a tab model.

The desired dispatch rule is:

```text
context-menu action
        │
        ├─ foreground process is Herdr
        │       → create a Herdr tab
        │
        └─ anything else
                → preserve native WezTerm behavior
```

This is deliberately asymmetric. The external multiplexer owns the nested workspace only while it is active; ordinary terminal panes should remain ordinary.

## Detect the active control plane

WezTerm exposes the foreground executable for a local pane:

```lua
local function pane_is_herdr(pane)
  local ok, process = pcall(function()
    return pane:get_foreground_process_name()
  end)

  return ok
    and process
    and process:lower():match('[\\/]herdr%.exe$') ~= nil
end
```

The `pcall` matters. A configuration callback should fail closed: if WezTerm cannot determine the foreground process, use the existing native behavior rather than breaking the context menu.

The action then becomes a small router:

```lua
if pane_is_herdr(pane) then
  spawn_herdr({
    'agent',
    '--kind', 'omp',
    '--label', 'jeeves',
    '--cwd', agent_directory,
  }, selected_text)
else
  local _, agent_pane = window:mux_window():spawn_tab {
    cwd = agent_directory,
    args = { 'omp.exe', '--cwd', agent_directory },
  }
  agent_pane:send_text(selected_text)
end
```

The fallback is the old implementation verbatim. That is an important constraint: adding Herdr awareness must not alter behavior when Herdr is absent.

## Let Herdr create Herdr objects

The Herdr side uses its own CLI as the control API:

```text
herdr workspace list
herdr tab create --workspace <workspace> --cwd <directory> --label <label>
herdr agent start <name> --kind omp --pane <pane-id>
herdr pane send-text <pane-id> <text>
```

The orchestration helper performs four steps:

1. Query the focused Herdr workspace.
2. Create a tab in that workspace and capture its root pane ID.
3. Start the requested agent in that pane.
4. Insert the selected text into the pane.

The workspace query is not cosmetic. A CLI process launched by WezTerm does not inherit the nested Herdr pane's context. Calling `herdr tab create` without an explicit workspace can therefore be ambiguous. Resolve the focused workspace first and pass its ID explicitly.

## Preserve interaction semantics, not just data

My first implementation used:

```text
herdr agent prompt <agent> <text>
```

It delivered the correct text—and immediately submitted it.

The old WezTerm implementation used `pane:send_text()`, which only filled the editor. I could inspect the selected stack trace, amend the request, add context, and press Enter when ready.

Those operations are not equivalent:

| Operation | Inserts text | Presses Enter | User can edit first |
|---|---:|---:|---:|
| WezTerm `pane:send_text()` | yes | no | yes |
| Herdr `agent prompt` | yes | yes | no |
| Herdr `pane send-text` | yes | no | yes |

The correct Herdr analogue was therefore `pane send-text`, not `agent prompt`.

This is the broader lesson: an integration can preserve payloads while breaking interaction. When replacing one API with another, compare the complete user-visible contract—focus, ownership, submission, working directory, and editability—not only the string that arrives.

## A path is not a command

A second bug hid behind a menu label: **Open in new tab (cd)**.

The implementation launched PowerShell with the selected directory as `-Command`:

```text
C:\work\project
```

PowerShell quite reasonably tried to execute the path as a cmdlet.

The label already specified the intended operation. The selected path should become the tab's working directory:

```lua
local path = resolve_path(selected_text, pane)

if pane_is_herdr(pane) then
  spawn_herdr({ 'run', '--label', 'shell', '--cwd', path })
else
  window:mux_window():spawn_tab { cwd = path }
end
```

No command is required. Both control planes now implement the same semantic action: create a shell tab whose current directory is the selected directory.

## Transport selected text without quoting it

Selected text can contain quotes, newlines, backticks, shell operators, and arbitrary diagnostic output. Putting it directly into a command-line argument creates two problems:

- shell quoting can change the payload;
- command-line inspection can expose text to unrelated processes.

The WezTerm callback writes the selection to a temporary UTF-8 file and passes only the filename to the helper. The helper reads the file, deletes it immediately, and sends the exact contents through Herdr's API.

```text
selection
   → temporary UTF-8 file
   → detached orchestration helper
   → read and delete file
   → Herdr pane send-text
```

The helper runs through `wezterm.background_child_process()`, keeping CLI startup and agent readiness waits off WezTerm's UI callback.

## Verification

The implementation was checked at three layers:

| Layer | Check |
|---|---|
| Configuration | Load the complete WezTerm config and enumerate actions without a Lua error |
| Helper | Six tests covering response parsing, focused-workspace selection, exact text transport, temp-file deletion, agent startup, and cwd-only shell tabs |
| Live behavior | Invoke the real context-menu action inside Herdr; confirm a Herdr tab appears, the selected prompt remains editable, and a cwd-only tab reports the selected directory |

The live test matters. Testing the helper directly proves the transport, but it does not prove that the WezTerm callback detected Herdr and selected the correct branch.

## Prior art and the remaining limitation

The ingredients are established:

- WezTerm users already implement [application-dependent keybindings](https://github.com/wez/wezterm/discussions/6184) with `get_foreground_process_name()`.
- WezTerm has its own [multiplexer and spawn APIs](https://wezterm.org/multiplexing.html).
- External multiplexers expose command interfaces; Herdr exposes tabs, panes, agents, and workspace operations through its CLI.
- WezTerm documents [OSC 1337 user variables](https://wezterm.org/recipes/passing-data.html) for explicit pane-to-Lua signaling.

I did not find an existing example that combines these pieces specifically for routing WezTerm context-menu actions into Herdr. That is not a novelty claim; it is the boundary of the search.

Foreground-process detection is also a local convenience, not a universal protocol. WezTerm's process inspection may be unavailable through remote or multiplexed domains and can be less reliable on Windows. A stronger future contract would have Herdr emit an explicit user variable such as:

```text
HERDR_ACTIVE=1
HERDR_WORKSPACE_ID=wJ
```

WezTerm could then read `pane:get_user_vars()` rather than infer ownership from a process name. Explicit state beats process archaeology.

## The pattern

This problem generalizes beyond Herdr:

```text
host terminal action
        ↓
detect active nested control plane
        ↓
route through the owner's API
        ↓
preserve the original interaction contract
```

When software is nested, creating the right object through the wrong owner is still wrong. Integrations should dispatch to the layer that owns lifecycle, navigation, and state—and preserve the small human affordances, such as editing before submission, that made the original workflow useful.

## Sources

- [Herdr](https://github.com/herdrdev/herdr)
- [WezTerm multiplexing](https://wezterm.org/multiplexing.html)
- [WezTerm `pane:get_foreground_process_name()`](https://wezterm.org/config/lua/pane/get_foreground_process_name.html)
- [WezTerm: Passing Data from a Pane to Lua](https://wezterm.org/recipes/passing-data.html)
- [Application-dependent WezTerm keybindings](https://github.com/wez/wezterm/discussions/6184)
- [WezTerm tmux control-mode proposal](https://github.com/wez/wezterm/issues/336)
- [Foreground process information limitations in mux/SSH mode](https://github.com/wez/wezterm/issues/6027)
