---
layout: post
title: "Right-click context menu in WezTerm: search, VS Code, Explorer from selected text"
date: 2026-05-15
categories: tools terminal windows
---

Windows Terminal lets you right-click to get a menu. But the menu is fixed — Copy, Paste, Find, Web Search (Preview only). You cannot add "Open in VS Code" or "Open in Explorer" based on what you've selected, because Windows Terminal has no `{selectedText}` variable for arbitrary actions.

WezTerm does. Here's the config.

## The problem with Windows Terminal

The `sendInput` action in `settings.json` types text into the terminal — it cannot read the current selection. The `searchWeb` action is the only built-in that consumes selected text, and it is gated behind the Preview channel.

## WezTerm's primitive: `get_selection_text_for_pane`

WezTerm's Lua API exposes `window:get_selection_text_for_pane(pane)` — the selected text as a string, available in any callback. This unlocks arbitrary actions on selections.

The right-click menu is built with `InputSelector` (WezTerm's fuzzy picker) triggered from a mouse binding:

```lua
local wezterm = require 'wezterm'
local act     = wezterm.action

-- resolve relative paths against the pane's working directory
local function resolve_path(sel, pane)
  if sel:match('^[A-Za-z]:[/\\]') or sel:match('^\\\\') or sel:match('^/') then
    return sel  -- already absolute
  end
  local cwd_url = pane:get_current_working_dir()
  if not cwd_url then return sel end
  local cwd = type(cwd_url) == 'string'
    and cwd_url:gsub('^file:///([A-Za-z]:)', '%1'):gsub('/', '\\')
    or  (cwd_url.file_path or tostring(cwd_url))
  return cwd:gsub('[/\\]+$', '') .. '\\' .. sel
end

local function url_encode(s)
  return (s:gsub('[^%w%-%.%_%~ ]', function(c)
    return string.format('%%%02X', string.byte(c))
  end):gsub(' ', '+'))
end

local function on_right_click(window, pane)
  local sel = window:get_selection_text_for_pane(pane):match('^%s*(.-)%s*$')

  if sel == '' then
    window:perform_action(act.PasteFrom 'Clipboard', pane)
    return
  end

  local preview = #sel > 50 and (sel:sub(1, 50) .. '…') or sel

  window:perform_action(
    act.InputSelector {
      title  = '› ' .. preview,
      fuzzy  = true,   -- type 's', 'v', 'e', 'c' → instant filter
      choices = {
        { label = '🔍  Search Google'    },
        { label = '📂  Open in VS Code'  },
        { label = '📁  Open in Explorer' },
        { label = '📋  Copy to clipboard' },
      },
      action = wezterm.action_callback(function(w, p, _id, label)
        if not label then return end
        if label:find('Search') then
          wezterm.open_with('https://www.google.com/search?q=' .. url_encode(sel))
        elseif label:find('VS Code') then
          wezterm.background_child_process({ 'cmd.exe', '/c', 'code', resolve_path(sel, p) })
        elseif label:find('Explorer') then
          wezterm.background_child_process({ 'explorer.exe', resolve_path(sel, p) })
        elseif label:find('Copy') then
          w:perform_action(act.CopyTo 'Clipboard', p)
        end
      end),
    },
    pane
  )
end
```

Wire it to right-click in `mouse_bindings`. Use `Up` not `Down` — `Down` shows the overlay while the button is still held, then `Up` dismisses it before you can select anything:

```lua
config.mouse_bindings = {
  -- copy on select (equivalent to WT's copyOnSelect: true)
  {
    event  = { Up = { streak = 1, button = 'Left' } },
    mods   = '',
    action = act.CompleteSelectionOrOpenLinkAtMouseCursor 'ClipboardAndPrimarySelection',
  },
  -- right-click: menu when text selected, paste otherwise
  {
    event  = { Up = { streak = 1, button = 'Right' } },
    mods   = '',
    action = wezterm.action_callback(on_right_click),
  },
}
```

The `CompleteSelectionOrOpenLinkAtMouseCursor` binding on left-click also handles URLs — clicking a link with no selection opens it directly, no Ctrl needed.

## Path resolution

`wezterm.background_child_process` spawns from WezTerm's install directory, not the terminal's working directory. The `resolve_path` function checks if the selection looks absolute (drive letter, UNC, Unix `/`). If not, it reads `pane:get_current_working_dir()` and prepends it. This makes selecting `src/main.py` from `ls` output and opening it in VS Code work correctly.

`get_current_working_dir()` returns a URL object in builds from 2024+. The function handles both the older string form (`file:///C:/...`) and the newer object form (`.file_path`).

## Comparison

| Feature | Windows Terminal | WezTerm |
|---|---|---|
| Right-click menu | ✅ built-in | ✅ via Lua |
| Web search on selection | ✅ Preview only | ✅ |
| Open selected path in VS Code | ❌ no `{selectedText}` | ✅ |
| Open selected path in Explorer | ❌ | ✅ |
| Relative path resolution | — | ✅ via `get_current_working_dir()` |
| Click URL (no Ctrl) | ❌ | ✅ side-effect of copy-on-select binding |
| Config language | JSON | Lua |

## Install

```powershell
scoop bucket add extras
scoop install wezterm
```

Config goes in `~\.wezterm.lua`. WezTerm hot-reloads on save — no restart needed while iterating.

Full config: [ankitg12/dotfiles — .wezterm.lua](https://github.com/ankitg12/dotfiles)

## Sources

- [WezTerm mouse binding docs](https://wezterm.org/config/mouse.html)
- [get_selection_text_for_pane](https://wezterm.org/config/lua/window/get_selection_text_for_pane.html)
- [InputSelector](https://wezterm.org/config/lua/keyassignment/InputSelector.html)
- [awesome-wezterm](https://github.com/michaelbrusegard/awesome-wezterm)
