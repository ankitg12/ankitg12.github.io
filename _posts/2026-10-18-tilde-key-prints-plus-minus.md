---
layout: post
title: "The tilde key prints ± — input source was the wrong layer"
date: 2026-10-18
categories: macos keyboard
---

**Problem**

The key left of `1` printed `§` and Shift printed `±`. Expected `` ` `` and `~`. The input menu already said U.S. Unplugging the keyboard and plugging it back in, before changing anything else, did nothing.

Two layers were wrong. Fixing only the first one left the key broken.

**What was wrong**

| Layer | What it controls | This machine |
|---|---|---|
| Input source | Which character a virtual key produces | `ABC-India`, then U.S. |
| Keyboard type | Which physical key is grave vs the ISO section key | Evofox cached as `41` (ISO) |

`ABC-India` is enough on its own to turn Shift+grave into `±`. Switching the input source to U.S. (`com.apple.keylayout.US`) did not fix it. The Evofox was still typed as ISO, so macOS treated the top-left key as the section key. U.S. layout maps that key to `§` / `±`.

The Evofox shows up as a generic USB keyboard, not under its brand name:

```text
VendorID  0x258a  (9610)
ProductID 0x013b  (315)
Product   USB Keyboard
```

`LMGetKbdType` returned `41`. The cache that survives replug is `/Library/Preferences/com.apple.keyboardtype.plist`. The key format is `ProductID-VendorID-0`:

```text
"315-9610-0" = 41;   # ISO
```

`40` is ANSI. `41` is ISO. `42` is JIS. Every other keyboard in that file was already `40`. Only the Evofox was `41`.

`macism` could not select U.S. while it was disabled (`Input source com.apple.keylayout.US does not exist`). It only selects sources that are already enabled. Enabling is a separate Text Input Source call. After U.S. was enabled, `macism` reported `com.apple.keylayout.US` and the key still printed `±`.

**What worked**

```bash
sudo defaults write /Library/Preferences/com.apple.keyboardtype.plist \
  keyboardtype -dict-add "315-9610-0" -int 40
```

That write does not retarget the live driver. The key kept printing `§` / `±` until the Evofox was unplugged and plugged back in. After that, the same key printed `~`.

Do not stack a `hidutil` swap of `0x700000064` and `0x700000035` on top. That remap is for a keyboard whose type you cannot change. Applied after type `40` has taken effect, it swaps the key back to `§`.

**Check**

```bash
hidutil list | grep 258a
python3 -c 'import ctypes; lib=ctypes.CDLL("/System/Library/Frameworks/Carbon.framework/Frameworks/HIToolbox.framework/HIToolbox"); lib.LMGetKbdType.restype=ctypes.c_uint8; print(lib.LMGetKbdType())'
defaults read /Library/Preferences/com.apple.keyboardtype.plist keyboardtype
```

`40` in the plist and `~` from the key. `LMGetKbdType` stays `41` until the device is reattached.

## Source

- [Specify a keyboard type on Mac](https://support.apple.com/guide/mac-help/specify-a-keyboard-type-mchlp2886/mac)
- [TN2450: Remapping Keys in macOS](https://developer.apple.com/library/archive/technotes/tn2450/_index.html) — the remap to skip once the type is correct
- [macism](https://github.com/laishulu/macism) — selects an enabled input source; does not enable one
