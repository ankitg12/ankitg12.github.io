---
layout: post
title: "Old hands, familiar keys: Cmd+Enter and window moves on macOS"
date: 2026-10-22
categories: macos productivity
---

Twenty years of Super+arrows does things to your fingers. New Mac, big external monitor, and every window move costs a thought. Voice dictation handles the typing now — but window placement is still muscle, and my muscle speaks Linux.

The setup I wanted: `Cmd+Enter` to toggle fullscreen, `Cmd+Left/Right` to throw the window across monitors. Three keys, no thinking. Getting there took three wrong layers before the right one.

## What I run

Tern (agent-first terminal), one laptop panel plus a portrait external, Rectangle for window moves, Karabiner-Elements for key remaps, MonitorControl for brightness. macOS 27, FileVault on.

## Wrong layer 1: Rectangle can't touch Apple-fullscreen

First try: bind it all in [Rectangle](https://github.com/rxhanson/Rectangle). `Cmd+Left/Right` to prev/next display worked. `Cmd+Enter` to restore did nothing — silently. No error, no hint.

Reason: I wrote `restore` into the plist. Not a real Rectangle action name. Rectangle ignores unknown keys without a word. And the window I was testing on was in macOS-native fullscreen (own Space), which Rectangle cannot manage at all. Two failures stacked to look like one.

## Wrong layer 2: the tooling lied too

Debugging produced its own comedy: `pgrep -x Rectangle` reported not-running while it was running (process name mismatch — plain `pgrep -fl` told the truth). `plutil -lint` rejected the Karabiner config as corrupt — wrong tool, Karabiner uses JSON, `jq empty` said VALID. I fixed a healthy file twice before checking the validator.

## Right layer: Karabiner for keys, Rectangle for windows

The split that works:

- **[Karabiner-Elements](https://github.com/pqrs-org/Karabiner-Elements)** (`~/.config/karabiner/karabiner.json`): `Cmd+Enter` → `Cmd+Ctrl+F`, the macOS fullscreen toggle. Karabiner reloads config automatically.

```json
{
    "description": "Cmd+Enter = Cmd+Ctrl+F (toggle fullscreen)",
    "from": { "key_code": "return_or_enter",
              "modifiers": { "mandatory": ["left_command"] } },
    "to": [{ "key_code": "f",
             "modifiers": ["left_command", "left_control"] }],
    "type": "basic"
}
```

- **Rectangle**: `Cmd+Left/Right` = prev/next display (Cmd-only `modifierFlags` in the plist). Karabiner remaps keys; it cannot move windows. Rectangle stays.

The lesson, stated plainly: Apple-fullscreen windows live on their own Space, unreachable by any window manager. Remap the toggle at the key layer, move windows at the window layer. One tool per layer.

## The price, paid deliberately

`Cmd+Left/Right` is line-start/line-end in every Mac text field. I sacrificed it. Voice input plus agent chat means I barely type anymore — three memorized keys beat Emacs-correct ones I never press. If you live in an editor, don't take this deal; put the moves on `Ctrl` instead.

Also worth knowing: base Rectangle is free MIT — the Pro upsell is a separate product. And macOS-fullscreen must exit (`Cmd+Enter` now) before a cross-display move; a window on its own Space has nowhere to go.

## Source

- [Rectangle](https://github.com/rxhanson/Rectangle) (MIT) — display moves
- [Karabiner-Elements](https://github.com/pqrs-org/Karabiner-Elements) — key remap
- Machine notes: `~/notes/machines/mac.md`, uniform-keys rule in `~/notes/machines/bootstrap.md`
