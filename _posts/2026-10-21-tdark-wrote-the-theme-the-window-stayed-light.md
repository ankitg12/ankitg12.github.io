---
layout: post
title: "tdark Wrote the Theme. The Window Stayed Light."
date: 2026-10-21 19:55:00 +0530
categories: terminal productivity
series: "Agent Mishaps"
---

`tdark` returned success. The window stayed light. Reloading the terminal applied light again.

The command had done two things that do not paint a [Tern](https://docs.stencil.so/tern/) window. It wrote a theme name into a marker file, and it emitted [OSC 11 / OSC 12](https://invisible-island.net/xterm/ctlseqs/ctlseqs.html) to recolor the background and the cursor. OSC only works in a terminal that consumes it. Tern's live state still had both appearance pins set to the light theme. A reload rereads those pins. It cannot invent a dark theme that was never stored there.

This is the same class of miss as [hiding Tern's top bar]({% post_url 2026-10-21-tern-topbar-autohide-css %}): the control you reached is not the control that draws the window.

## Two pins, one appearance

Tern keeps `theme_dark` and `theme_light`. The script surface is `theme [dark|light] "<name>"`: one argument is always a theme name, and changing one appearance preserves the other. While the machine is in light mode, a script that only updates the dark slot changes nothing you can see.

Pinning both slots to the same name is the explicit switch. `tdark` means dark now, not "dark when the OS is dark."

`tern ctl state` is the check. After the failed `tdark`, both fields were still `light-gruvbox`.

## The live write is not the file

Writing `settings.json` and reloading did not help, because the file still said light. The live path is a window plugin calling `settings:set` on both pins:

```lua
cx.settings:set("theme_light", theme)
cx.settings:set("theme_dark", theme)
```

The plugin polls the marker the theme command already writes. Once it is loaded, `tdark` and `tlight` do not need to know about Tern. They write the file. The window follows within one poll.

## The colon is the method call

Luau inherits Lua's rule: `obj:method(a)` is sugar for `obj.method(obj, a)`, with `obj` evaluated once. A dot call does not pass `self`.

```lua
cx.settings.set("theme_light", theme) -- theme name arrives as self
cx.settings:set("theme_light", theme) -- SettingsCx arrives as self
```

The dot form produced this, once per poll:

```
bad argument `self` to `SettingsCx.set`:
error converting Lua string to userdata (expected userdata of type 'SettingsCx')
```

`tern plugin list` still said Ready. A failed window entry does not flip that catalog. Preferences shows the worse of the host status and the window status, and the first line of the error is in the load toast and in `~/Library/Logs/Tern/tern.log`.

## Reload does not re-run window_start

`window_start` runs once per plugin per window, after the session has settled. A later `tern plugin reload` drops the window VMs and runs each entry again, but it does not re-fire `window_start` for a plugin that already ran it. A watcher registered only there never starts after a hot reload of a plugin you already loaded.

Start it from the entry as well:

```lua
local function tick(cx)
  apply(cx, read_marker())
  tern.timer(200, tick)
end

tern.on("window_start", function(cx)
  tick(cx)
end)

-- plugin reload does not re-fire window_start
tern.timer(50, tick)
```

`tern plugin link` hot-reloads. One reload loads the plugin. After that, the theme command is just a file write.

A missing `end` fails the same way: the window entry does not load, the toast says so, and `tern plugin list` still prints Ready. The log line is the one to read.

## What actually changed

Same window. No terminal restart.

| Step | `theme_dark` | `theme_light` |
|---|---|---|
| Marker set to `dark-volcanic` | `dark-volcanic` | `dark-volcanic` |
| `tlight` | `light-gruvbox` | `light-gruvbox` |

The agent UI theme is a separate watcher on the same marker. It does not paint the terminal. If that watcher is not loaded, the chrome can go dark while the UI stays light, or the reverse.

## Source

- [Tern plugin lifecycle](https://docs.stencil.so/tern/concepts/lifecycle.html) — `window_start` once per plugin per window; window-entry failure vs `tern plugin list`
- [Tern terminal scripts](https://docs.stencil.so/tern/scripts/terminal.html) — `theme [dark|light] "<name>"`, and `theme_dark` / `theme_light` in `tern ctl state`
- [Lua 5.1 §2.5.9](https://www.lua.org/manual/5.1/manual.html#2.5.9) — colon calls pass `self`
- [XTerm control sequences](https://invisible-island.net/xterm/ctlseqs/ctlseqs.html) — OSC 11 and OSC 12
