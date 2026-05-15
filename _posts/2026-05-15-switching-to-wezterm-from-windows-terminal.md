---
layout: post
title: "Switching to WezTerm from Windows Terminal"
date: 2026-05-15
categories: tools terminal windows
---

I assumed Windows Terminal would be the best terminal on Windows. Microsoft built it, it ships with Windows 11, it gets regular updates, it has an active GitHub. When a company with those resources is maintaining a terminal, what would a side project add?

Turns out: a lot.

## What I wanted

Select text in the terminal → right-click → pick an action: search Google, open the path in VS Code, open it in Explorer. The kind of thing you take for granted in a browser or IDE.

## What Windows Terminal actually offers

Windows Terminal's right-click context menu has Copy, Paste, Find, and — in the Preview channel only — Web Search. You cannot extend it with custom actions on selected text.

The reason: `sendInput` (WT's action for typing into the terminal) has no `{selectedText}` variable. The only action that consumes a selection is `searchWeb`, and it is Preview-gated. There is no supported path to "run arbitrary command on selected text" in WT's `settings.json`.

I filed this away as a limitation and moved on. For a while.

## The switch

WezTerm is configured in Lua. Lua means a real programming language, which means you can read terminal state. The specific function: `window:get_selection_text_for_pane(pane)` — returns the selected text as a string, available in any callback.

```lua
local function on_right_click(window, pane)
  local sel = window:get_selection_text_for_pane(pane):match('^%s*(.-)%s*$')

  if sel == '' then
    window:perform_action(act.PasteFrom 'Clipboard', pane)
    return
  end

  window:perform_action(
    act.InputSelector {
      title  = '› ' .. (sel:sub(1, 50)),
      fuzzy  = true,
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

`fuzzy = true` means you type `s` → Enter for Search, `v` → Enter for VS Code. No mouse needed after right-click.

One non-obvious detail: bind the right-click to `Up`, not `Down`. If you bind `Down`, the overlay appears while the button is still held — the `Up` event then dismisses it before you can select anything.

```lua
config.mouse_bindings = {
  {
    event  = { Up = { streak = 1, button = 'Right' } },
    mods   = '',
    action = wezterm.action_callback(on_right_click),
  },
}
```

## Relative paths

`wezterm.background_child_process` spawns from WezTerm's install directory. If you select `src/main.py` from `ls` output and open it in VS Code, it will look in `C:\Program Files\WezTerm\src\main.py`.

Fix: check if the selection is already absolute, and if not, read the pane's working directory and prepend it.

```lua
local function resolve_path(sel, pane)
  if sel:match('^[A-Za-z]:[/\\]') or sel:match('^\\\\') or sel:match('^/') then
    return sel
  end
  local cwd_url = pane:get_current_working_dir()
  if not cwd_url then return sel end
  local cwd = type(cwd_url) == 'string'
    and cwd_url:gsub('^file:///([A-Za-z]:)', '%1'):gsub('/', '\\')
    or  (cwd_url.file_path or tostring(cwd_url))
  return cwd:gsub('[/\\]+$', '') .. '\\' .. sel
end
```

## What else changed

Moving from WT's `settings.json` to a Lua file meant porting everything:

- Font and color scheme defined inline with exact hex values (ported from WT's `GruvboxLight` scheme)
- Launch menu replaces WT profiles — named entries with starting directories, including project-specific ones
- `ctrl+v` for paste, `alt+shift+d` for split (WT defaults preserved)
- Copy on select via `CompleteSelectionOrOpenLinkAtMouseCursor 'ClipboardAndPrimarySelection'` — also handles URL clicks as a side-effect, so plain click on a URL opens it, no Ctrl needed

## What Windows Terminal is better at

To be fair: Windows Terminal integrates with Windows as a default terminal handler (`DelegationTerminal`). When you double-click a `.bat` file or a console app opens, it can appear in Windows Terminal. WezTerm does not implement this COM interface ([issue #7534](https://github.com/wezterm/wezterm/issues/7534), open as of early 2026). So Windows Terminal remains the system default; WezTerm is what I open manually.

## The broader point

Microsoft's resources go into Windows Terminal being a good default for everyone. WezTerm's resources go into making it configurable for anyone who wants to go further. These are different goals, and Windows Terminal is genuinely good at its goal. I just needed the other thing.

## Setup

```powershell
scoop bucket add extras
scoop install wezterm
```

Config: `~\.wezterm.lua`. Hot-reloads on save.

Full config: [ankitg12/dotfiles — .wezterm.lua](https://github.com/ankitg12/dotfiles)

## Sources

- [WezTerm mouse bindings](https://wezterm.org/config/mouse.html)
- [get_selection_text_for_pane](https://wezterm.org/config/lua/window/get_selection_text_for_pane.html)
- [awesome-wezterm](https://github.com/michaelbrusegard/awesome-wezterm) — plugin directory
- [Session management in WezTerm without tmux](https://fredrikaverpil.github.io/blog/2024/10/20/session-management-in-wezterm-without-tmux/)
