---
layout: post
title: "Porting AutoHotkey to Wayland: The Mechanics of Turning a Volume Knob into a Scroll Wheel"
date: 2026-09-18
categories: linux wayland productivity
series: "Moving to Linux"
---

I recently moved my primary workstation from Windows to Linux. Within an hour, I hit a physical workflow friction I couldn't live without: my keyboard's volume knob no longer scrolled.

When I originally set this up on Windows (alongside other workstation ergonomics like [AutoHotkey shortcuts for terminal tooling]({% post_url 2026-06-05-screenshot-claude-code-windows %})), the wheel-scroll script was so compact—literally five lines—that it was almost too trivial to write about:

```ahk
#Requires AutoHotkey v2.0
#SingleInstance Force

SCROLL_SPEED := 5
Volume_Up::   Send("{WheelUp "   SCROLL_SPEED "}")
Volume_Down:: Send("{WheelDown " SCROLL_SPEED "}")
```

On Linux Wayland, those five lines exploded into a deep architectural dive across kernel input topologies, `/dev/uinput` security boundaries, `libinput` pointer device classification, and high-resolution scroll detent arithmetic.

This post documents why remapping keys is straightforward on Windows, why Wayland's security model deliberately breaks that simplicity, and the three subtle kernel/compositor gotchas you hit when building a modern Linux replacement.

## The Architectural Mismatch: Global Hooks vs. Zero-Trust Compositors

On Windows, the input subsystem exposes a single desktop message queue:

| Dimension | Windows (Win32 / AHK) | Linux (Wayland / Mutter) |
|---|---|---|
| **Hook Mechanism** | `SetWindowsHookEx(WH_KEYBOARD_LL)` | Kernel `evdev` (`/dev/input/event*`) |
| **Injection Mechanism** | `SendInput()` | Kernel `uinput` (`/dev/uinput`) |
| **Privilege Model** | Standard unprivileged user process | Root daemon or `input`/`uinput` group |
| **Compositor Role** | Transparent message forwarder | Strict window isolation barrier |
| **Scroll Unit** | Discrete ticks (1 detent = 1 notch) | High-res ticks (`REL_WHEEL_HI_RES`, 120 units/detent) |

On Windows, any unprivileged background process can register a low-level keyboard hook. The OS happily yields raw keystrokes from any window, lets you suppress them, and allows `SendInput()` to synthesize mouse wheel clicks into whatever application holds focus. No root privileges, no kernel drivers, no background system services.

Wayland treats that exact design as a critical security vulnerability.

Under Wayland, client applications are isolated. A process cannot inspect keypresses intended for another window (preventing keyloggers) and cannot synthesize input events into another window's surface (preventing input injection). Consequently, global hardware remapping cannot exist at the compositor or display server client level. It must occur below the display server, directly inside the Linux kernel input subsystem:

```text
[ Physical Keyboard ]
        │
        ▼
  /dev/input/event*  (Kernel evdev — requires 'input' group)
        │
        ▼
  [ Remapping Daemon ]  (Grabs raw hardware events, intercepts volume keys)
        │
        ▼
   /dev/uinput       (Kernel uinput — creates virtual pointer/keyboard)
        │
        ▼
 [ libinput / Mutter ] (Wayland compositor filters & routes to focused window)
```

## Trap 1: The Multi-Interface USB Topology

When you plug in a modern mechanical keyboard with a rotary encoder, the kernel does not register a single input node. It creates a cluster of distinct event nodes under `/proc/bus/input/devices`:

```text
/dev/input/event12: "USB Keyboard" (Typing keys, EV_KEY 1..248)
/dev/input/event14: "USB Keyboard Consumer Control" (Volume knob, Mute, Media)
/dev/input/event15: "USB Keyboard System Control" (Sleep, Power)
```

The rotary knob is almost always wired to the **Consumer Control** HID interface (`event14`), not the primary typing keyboard (`event12`).

If an input grabber attaches only to the main keyboard device, it never receives `KEY_VOLUMEUP` or `KEY_VOLUMEDOWN`. Any robust solution must either:
1. Scan all input nodes matching the keyboard's USB Vendor/Product ID and attach to the specific interface advertising `KEY_VOLUMEUP`.
2. Inspect the bus type (`BUS_USB = 0x03`, `BUS_BLUETOOTH = 0x05`) to intercept external keyboard wheels while leaving internal laptop volume keys (`BUS_I8042 = 0x11`, `BUS_HOST = 0x19`) untouched.

## Trap 2: The libinput Pointer Classification Gate

Once you read the raw volume events, you must emit mouse scroll events into `/dev/uinput`.

However, if your virtual device only advertises keyboard capabilities and relative wheel motion (`EV_REL: [REL_WHEEL]`), modern Wayland compositors (GNOME/Mutter, Sway) will **silently discard every scroll event**.

`libinput` classifies input devices based on their declared capabilities. If a virtual device emits scroll events but does not advertise pointer buttons (`BTN_LEFT`, `BTN_RIGHT`), `libinput` tags the device as a non-pointer:

```text
libinput error: device 'Virtual Keyboard' has scroll axes but no pointer buttons.
Ignoring relative wheel events from non-pointer device.
```

To make Wayland accept the scroll events, the virtual device descriptor in `uinput` must declare at least one mouse button (`EV_KEY: [BTN_LEFT]`) or be declared as a composite `keyboard + mouse` device.

## Trap 3: The High-Resolution Scroll Scale (`REL_WHEEL_HI_RES`)

In traditional X11 and AutoHotkey, scrolling is discrete: 1 notch on the wheel equals 1 detent, and `SCROLL_SPEED := 5` sends 5 distinct clicks.

Modern Linux compositors use **high-resolution wheel scrolling**. In `libinput`, a standard physical detent is defined as **120 units** of `REL_WHEEL_HI_RES`:

$$\text{1 Physical Click} = 120 \times \text{REL\_WHEEL\_HI\_RES} = 1 \times \text{REL\_WHEEL}$$

When using remapping tools like [input-remapper](https://github.com/sezanzeb/input-remapper), the internal macro engine scales wheel speed using this exact ratio:

```python
# inputremapper/injection/macros/macro.py
code, value = {
    "up": ([REL_WHEEL, REL_WHEEL_HI_RES], [1 / 120, 1]),
    "down": ([REL_WHEEL, REL_WHEEL_HI_RES], [-1 / 120, -1]),
}[direction.lower()]

for i in range(0, 2):
    float_value = value[i] * resolved_speed + remainder[i]
    if abs(float_value) >= 1:
        handler(EV_REL, code[i], int(float_value))
```

If you specify `speed: 5` (copying your AutoHotkey config), the math evaluates to:

$$\text{REL\_WHEEL} = \frac{1}{120} \times 5 = 0.0416 \implies \text{int}(0.0416) = 0$$

The driver emits **zero** legacy scroll notches and only 5 units of high-resolution scroll — exactly $\frac{1}{24}\text{th}$ of a single detent. The window barely moves.

To achieve AutoHotkey's 5-line scrolling speed, the speed parameter must be multiplied by the 120-unit base detent:

$$\text{Speed} = 5 \times 120 = 600$$

## The Working Configuration

To implement this reliably without writing a custom root daemon from scratch, [input-remapper](https://github.com/sezanzeb/input-remapper) provides an ideal architecture: its root service handles `/dev/uinput` device creation, while an unprivileged user CLI manages configuration presets via D-Bus.

Here is the complete preset configuration saved to `~/.config/input-remapper-2/presets/<Device_Name>/wheel-scroll.json`:

```json
[
    {
        "input_combination": [
            {
                "type": 1,
                "code": 115,
                "origin_hash": "YOUR_CONSUMER_CONTROL_HASH",
                "analog_threshold": null,
                "message_type": "selected_event"
            }
        ],
        "target_uinput": "keyboard + mouse",
        "output_symbol": "wheel(up, 600)",
        "mapping_type": "key_macro"
    },
    {
        "input_combination": [
            {
                "type": 1,
                "code": 114,
                "origin_hash": "YOUR_CONSUMER_CONTROL_HASH",
                "analog_threshold": null,
                "message_type": "selected_event"
            }
        ],
        "target_uinput": "keyboard + mouse",
        "output_symbol": "wheel(down, 600)",
        "mapping_type": "key_macro"
    }
]
```

To activate and persist across reboots:

```bash
# Set preset to load automatically when device is plugged in
cat <<EOF > ~/.config/input-remapper-2/config.json
{
    "autoload": {
        "Your Keyboard Name": "wheel-scroll"
    },
    "version": "2.0.1"
}
EOF

# Trigger immediate injection
input-remapper-control --command autoload
```

## Summary

Moving from Windows to Linux often feels like trading convenience for control. AutoHotkey makes simple things trivial because Windows leaves its input pipeline unsegmented. Linux Wayland forces you to understand the full stack — HID interfaces, kernel `evdev`, virtual `uinput` devices, and compositor scroll curves.

Once wired correctly, the result is cleaner: no user-space hooks polling your keystrokes, no fragile window title matching, and native hardware-level scrolling across every Wayland and XWayland surface.

## Source

- [AutoHotkey Source Repository](https://github.com/AutoHotkey/AutoHotkey)
- [Linux Kernel uinput Subsystem Documentation](https://docs.kernel.org/input/uinput.html)
- [libinput High-Resolution Scrolling Specification](https://wayland.freedesktop.org/libinput/doc/latest/scrolling.html)
- [input-remapper Daemon and Control Tool](https://github.com/sezanzeb/input-remapper)
- [python-evdev API Reference](https://python-evdev.readthedocs.io/)
