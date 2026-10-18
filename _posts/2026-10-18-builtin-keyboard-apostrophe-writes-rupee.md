---
layout: post
title: "The built-in keyboard apostrophe writes ₹ — U.S. was enabled, not selected"
date: 2026-10-18
categories: macos keyboard
---

**Problem**

The built-in keyboard's apostrophe key wrote `₹`. Selecting U.S. in System Settings did not change the key.

This is not the external-keyboard failure from earlier today. That one was an Evofox, typed as ISO, so the grave key printed `§` / `±`. This key is on the Apple Internal Keyboard / Trackpad. The other post is [The tilde key prints ±]({% post_url 2026-10-18-tilde-key-prints-plus-minus %}).

**What was wrong**

U.S. was enabled. ABC-India was still selected.

```text
AppleCurrentKeyboardLayoutInputSourceID = com.apple.keylayout.ABC-India

selected:
  KeyboardLayout ID = -3
  KeyboardLayout Name = ABC-India

enabled:
  ABC-India (ID -3)
  U.S. (ID 0)
```

A Settings click does not rewrite `AppleSelectedInputSources`. The input menu can list U.S. while ABC-India stays the selected layout. ABC-India maps the key that U.S. uses for `'` to `₹`.

The device is not the Evofox, and it is not on USB:

```text
VendorID   0x05ac
ProductID  0x0281
Product    Apple Internal Keyboard / Trackpad
Transport  SPI
Service    AppleHIDKeyboardEventDriverV2
```

Ruled out before changing the layout again:

| Check | Result |
|---|---|
| Karabiner-Elements | not running |
| `hidutil property --get UserKeyMapping` | `(null)` on every device, including this service |
| Text Replacements | `omw` → `On my way!` only |
| Evofox keyboard-type cache | different device. Not this key |

**What worked**

Enable, select, and disable are separate Text Input Source calls. Select U.S., then disable ABC-India so it cannot remain selected:

```swift
TISEnableInputSource(us)      // 0
TISSelectInputSource(us)      // 0
TISDisableInputSource(india)  // 0
```

`macism` only selects a source that is already enabled. Enabling U.S. was not the missing step. U.S. was already enabled. It was not selected.

**Check**

```bash
defaults read com.apple.HIToolbox AppleCurrentKeyboardLayoutInputSourceID
```

```text
com.apple.keylayout.US
```

ABC-India stays installed. It is disabled, so it is gone from `AppleEnabledInputSources` and from `AppleSelectedInputSources`.

## Source

- [Change Input Sources settings on Mac](https://support.apple.com/guide/mac-help/change-input-sources-settings-mchlp2864/mac) — the Settings control that did not commit the selected source
- [macism](https://github.com/laishulu/macism) — selects an enabled input source; does not enable one, and does not fix "enabled but not selected"
- [The tilde key prints ±](https://ankitg12.github.io/macos/keyboard/2026/10/18/tilde-key-prints-plus-minus.html) — external keyboard, keyboard-type cache, not this key
