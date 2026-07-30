---
layout: post
title: "The File Path Was Fine. The Invisible Hyperlink Target Wasn't"
date: 2026-07-30
categories: wezterm terminal ai agents productivity debugging
series: "AI coding agent productivity"
---

A file citation in my coding agent looked correct, turned purple on hover, and did absolutely nothing when clicked.

A plain PDF link opened normally. A tool-generated citation such as:

```text
C:/Users/<username>/.wezterm.lua:287-304
```

did not. The obvious suspects—file association, path separators, the displayed line range—were all plausible and all incomplete. The bug lived in text that was never displayed.

This is a follow-up to [Clickable file paths in your terminal: why they work in your AI agent but not your shell]({% post_url 2026-07-22-clickable-file-paths-wezterm-osc8-vs-hyperlink-rules %}). That post explains how OSC 8 hyperlinks differ from WezTerm's regex-based `hyperlink_rules`. This one is about the next layer: **the visible label can be correct while the invisible OSC 8 target is wrong for the action that consumes it.**

## What I saw was not what WezTerm opened

OSC 8 hyperlinks contain two independent values:

```text
visible label: C:/Users/<username>/.wezterm.lua:287-304
hidden target: file:///C:/Users/<username>/.wezterm.lua?line=287
```

The terminal renders the label but passes the target URI to its click handler. Looking only at the screen therefore led me toward stripping `:287-304` from the displayed path. That was the wrong evidence layer.

My `open-uri` handler decoded the target and called the operating system's default application:

```lua
wezterm.on('open-uri', function(_window, _pane, uri)
  local path = uri:match('^file:///?(.+)$')
  if not path then return end

  path = url_decode(path):gsub('/', '\\')
  wezterm.open_with(path)
  return false
end)
```

For an ordinary link, this is exactly what I wanted:

```text
file:///C:/Users/<username>/Downloads/guide.pdf
→ C:\Users\<username>\Downloads\guide.pdf
```

For the agent citation, it produced:

```text
file:///C:/Users/<username>/.wezterm.lua?line=287
→ C:\Users\<username>\.wezterm.lua?line=287
```

`wezterm.open_with` correctly handed that string to Windows. Windows correctly failed to find it: `?line=287` is source-location metadata, not part of the physical filename.

## Log the event boundary

The useful diagnostic was not another regex edit. It was two temporary log statements at the boundary where WezTerm receives and dispatches the click:

```lua
wezterm.on('open-uri', function(_window, _pane, uri)
  wezterm.log_info('link-click raw_uri=', uri)

  local path = uri:match('^file:///?(.+)$')
  if not path then return end

  path = menu_text_for_uri(uri)
  wezterm.log_info('link-open normalized_path=', path, ' action=open_with')
  wezterm.open_with(path)
  return false
end)
```

`wezterm.log_info` writes through WezTerm's logging layer. On Windows, the GUI logs were under:

```text
~/.local/share/wezterm/wezterm-gui.exe-log-*.txt
```

One click settled the question:

```text
link-click raw_uri= file:///C:/Users/<username>/.wezterm.lua?line=287
link-open normalized_path= C:\Users\<username>\.wezterm.lua?line=287 action=open_with
```

This proved four things at once:

1. The colored citation was a real hyperlink, not merely styled text.
2. WezTerm's `open-uri` event fired.
3. The agent encoded line information as a URI query parameter.
4. My normalizer passed that metadata into the filesystem path.

After the fix, the same click logged:

```text
link-click raw_uri= file:///C:/Users/<username>/.wezterm.lua?line=23
link-open normalized_path= C:\Users\<username>\.wezterm.lua action=open_with
```

The file opened. The temporary logging then came out; diagnostic instrumentation should not become permanent configuration by accident.

## The fix

Normalize file-link metadata before converting separators or calling the default opener:

```lua
local function menu_text_for_uri(uri)
  local path = uri:match('^file:///?(.+)$')
  if not path then return uri end

  path = url_decode(path)

  -- Agent citations append ?line=N or :selector;
  -- Windows needs only the physical path.
  path = path:gsub('[?#].*$', ''):gsub(':[^/\\]+$', '')

  if path:match('^~') then
    path = wezterm.home_dir .. path:sub(2)
  end

  return path:gsub('/', '\\')
end
```

Then keep the click handler boring:

```lua
wezterm.on('open-uri', function(_window, _pane, uri)
  local path = uri:match('^file:///?(.+)$')
  if not path then return end

  wezterm.open_with(menu_text_for_uri(uri))
  return false
end)
```

The order matters:

| Step | Why |
|---|---|
| Confirm `file://` | Do not rewrite web URLs or custom schemes |
| Percent-decode | Turn `%20` and other encoded bytes into the local path |
| Remove query/fragment | Drop URI metadata such as `?line=287` or `#L287` |
| Remove agent selector | Handle textual selectors such as `:287-304` or `:raw` |
| Expand `~` | Resolve the home-relative path locally |
| Convert separators | Produce the native Windows path |
| Call `open_with` | Let the OS choose PDF viewer, browser, Office, editor, and so on |

The broad `:[^/\\]+$` rule is safe in this handler because it runs only after the string has been recognized as a local `file://` target. It deliberately treats any final colon suffix as agent metadata. If you use NTFS alternate data streams (`file.txt:stream`), narrow the selector pattern instead.

## Preserve the line number when the editor is the destination

Stripping `?line=287` is correct when the goal is **open this file with its registered application**. It intentionally discards navigation metadata.

If every source file should open in an editor at the cited line, parse the query instead and call the editor's location-aware CLI. For VS Code:

```lua
local path, line = decoded:match('^(.-)%?line=(%d+)$')
if path and line then
  wezterm.background_child_process({ 'code', '--goto', path .. ':' .. line })
end
```

These are different product choices:

| Desired behavior | Correct action |
|---|---|
| Respect Windows file associations | Strip source metadata, then `wezterm.open_with(path)` |
| Always jump to source location in VS Code | Parse metadata, then `code --goto path:line` |
| Offer both | Default-open on left click; put editor-specific actions in a context menu |

I chose the third model: left click means the system default; the context menu retains explicit editor actions.

## The debugging lesson

A hyperlink has at least three relevant representations:

```text
what the user sees
        ↓
what OSC 8 declares as the target URI
        ↓
what the click handler passes to the operating system
```

A bug can exist between any two layers. Screenshot inspection covers only the first. Reading configuration covers intended behavior. Logging the event boundary exposes actual behavior.

The shortest reliable debugging loop was:

1. Add one log at event ingress.
2. Add one log immediately before dispatch.
3. Click once.
4. Compare the two strings.
5. Fix the transformation.
6. Click once to verify.
7. Remove the logs.

That is less work than repeatedly editing regexes against the text visible on screen—and it produces evidence instead of a plausible story.

## Source

- [WezTerm `open-uri` event](https://wezterm.org/config/lua/window-events/open-uri.html)
- [WezTerm `wezterm.open_with`](https://wezterm.org/config/lua/wezterm/open_with.html)
- [WezTerm `wezterm.log_info`](https://wezterm.org/config/lua/wezterm/log_info.html)
- [OSC 8 hyperlinks in terminal emulators](https://gist.github.com/egmontkob/eb114294efbcd5adb1944c9f3cb5feda)
