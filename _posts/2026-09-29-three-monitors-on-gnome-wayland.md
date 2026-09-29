---
layout: post
title: "Three Monitors on GNOME Wayland: Rotation, Scale and Brightness After a Reboot"
date: 2026-09-29 17:55:00 +0530
categories: linux wayland debugging
series: "Moving to Linux"
---

After one reboot, my desk broke in three ways at once: a portrait monitor was upside down, the 4K monitor had tiny text, and the brightness sliders for the external monitors were gone. GNOME Settings refused to fix the first problem and had no way to fix the third.

The desk is a laptop plus three external monitors: two 1080p panels in portrait orientation and one 27" 4K panel. They connect through a USB-C dock. The laptop runs Ubuntu 24.04 with GNOME 46 on an AMD Ryzen APU. Before the reboot I used an Xorg session with an `autorandr` profile. After it, I was on Wayland.

Each failure has a different root cause. None of them is "Linux is bad at displays".

## 1. The rotated monitor: two saved layouts, not one

### Symptom

One portrait monitor was upside down relative to the other. Settings → Displays showed "Portrait Right". Changing it to "Portrait Left" and clicking Apply gave this:

> **Changes Cannot be Applied** — This could be due to hardware limitations.

### Why autorandr did not help

`autorandr` saves and restores `xrandr` state. `xrandr` talks to the X server. On Wayland there is no X server that controls the outputs: the compositor (Mutter, for GNOME) owns the layout, and it reads its layout only from `~/.config/monitors.xml`. So an autorandr profile has no effect on Wayland.

### Why the orientation was wrong

`monitors.xml` does not store one layout per desk. It stores one layout per **set of connectors**, and the connector names are different in each session type:

| Session | Connector names on this laptop |
|---|---|
| Xorg (amdgpu DDX) | `DisplayPort-6`, `DisplayPort-7`, `DisplayPort-8` |
| Wayland (Mutter, KMS) | `DP-7`, `DP-8`, `DP-9` |

So the same four monitors had two independent entries in `monitors.xml`:

```xml
<!-- Wayland entry: old, wrong -->
<rotation>right</rotation>
<connector>DP-7</connector>
<vendor>TCL</vendor>

<!-- Xorg entry: fixed earlier today, correct -->
<rotation>left</rotation>
<connector>DisplayPort-6</connector>
<vendor>TCL</vendor>
```

The earlier fix in the Xorg session updated only the Xorg entry. After the reboot, Mutter loaded the stale Wayland entry.

### Why Settings refused, and what worked

The hardware can do the rotation. I sent the same change straight to Mutter through its D-Bus API, `org.gnome.Mutter.DisplayConfig.ApplyMonitorsConfig`, and it accepted it. That method takes a mode argument: `0` verifies the layout only, `1` applies it temporarily, and `2` applies it and saves it to `monitors.xml`.

```python
from gi.repository import Gio, GLib

bus = Gio.bus_get_sync(Gio.BusType.SESSION)
dc = Gio.DBusProxy.new_sync(bus, 0, None,
    "org.gnome.Mutter.DisplayConfig",
    "/org/gnome/Mutter/DisplayConfig",
    "org.gnome.Mutter.DisplayConfig")

serial, monitors, logical, _ = dc.call_sync("GetCurrentState", None, 0, -1).unpack()
current = {m[0][0]: next(md[0] for md in m[1] if md[6].get("is-current"))
           for m in monitors}

ROTATE = {"DP-7": 1}   # transform 1 = 90°; see the transform enum in the D-Bus XML
layout = [(x, y, scale, ROTATE.get(ms[0][0], t), primary,
           [(ms[0][0], current[ms[0][0]], {})])
          for x, y, scale, t, primary, ms, _ in logical]

sig = "(uua(iiduba(ssa{sv}))a{sv})"
dc.call_sync("ApplyMonitorsConfig", GLib.Variant(sig, (serial, 0, layout, {})), 0, -1)  # verify
dc.call_sync("ApplyMonitorsConfig", GLib.Variant(sig, (serial, 2, layout, {})), 0, -1)  # persist
```

I do not know exactly which layout Settings built. It is the Settings panel, not the hardware, that rejected the change.

### A CLI for next time: gnome-randr

There is no third-party GUI for display layout on GNOME Wayland, and there cannot really be one. Each Wayland compositor has its own private layout API. Tools such as `wdisplays`, `kanshi` and `way-displays` use `wlr-output-management`, which Mutter does not implement. GNOME's official CLI, `gdctl`, arrived in GNOME 48.

On GNOME 46, [`gnome-randr`](https://github.com/maxwellainatchi/gnome-randr-rust) is a small Rust CLI over the same D-Bus API:

```bash
sudo apt install libdbus-1-dev          # libdbus-sys needs the headers
CARGO_TARGET_DIR=~/.cache/cargo-target cargo install gnome-randr   # /tmp is noexec here

gnome-randr query
gnome-randr modify DP-7 --rotate right --persistent
```

Its rotation names do not match `monitors.xml`. For the same physical monitor, `gnome-randr query` reports `rotation: right`, while `monitors.xml` stores `<rotation>left</rotation>`. Use each tool's own names.

## 2. The tiny 4K text: scale, not resolution

The 4K panel ran at 3840×2160 with scale 1.0. Setting a lower resolution makes text bigger but blurry. The correct fix is to keep the native mode and set scale 2:

```bash
gnome-randr modify DP-8 --scale 2 --persistent
```

My first attempt also moved the other monitors to close the gap, and Mutter rejected it with `Logical monitors overlap`. This display uses GNOME's *physical* layout mode, where a scaled monitor keeps its full pixel size in the layout. Nothing needs to move.

With fractional scaling off, this mode offers only `x1.00, x2.00, x3.00, x4.00`. For 150% or 175%, enable fractional scaling:

```bash
gsettings set org.gnome.mutter experimental-features "['scale-monitor-framebuffer']"
```

## 3. The lost brightness sliders: a udev rule that misses the APU

External monitor brightness goes over DDC/CI, which is I²C carried through the display cable. The GNOME extension [Brightness control using ddcutil](https://github.com/daitj/gnome-display-brightness-ddcutil) calls `ddcutil`. After the reboot:

```
$ ddcutil detect --brief
Device /dev/i2c-23 is not readable and writable.  Error = EACCES(13): Permission denied
No displays found.
```

The devices are `root:i2c 0660`, and I am not in the `i2c` group. The `ddcutil` package includes a udev rule that gives the logged-in user access through `uaccess`:

```
# /usr/lib/udev/rules.d/60-ddcutil-i2c.rules
SUBSYSTEM=="i2c-dev", KERNEL=="i2c-[0-9]*", ATTRS{class}=="0x030000", TAG+="uaccess"
```

`0x030000` is the PCI class of a *VGA-compatible controller*. The APU reports a different class:

```
$ cat /sys/bus/pci/devices/0000:c5:00.0/class
0x038000
```

`0x038000` is *display controller, other*. The rule never matches, so no ACL is set. Before the reboot, access had come from a manual change that did not survive the reboot. A local rule that matches the whole display class `0x03xxxx` fixes it:

```bash
echo 'SUBSYSTEM=="i2c-dev", KERNEL=="i2c-[0-9]*", ATTRS{class}=="0x03[0-9a-f]*", TAG+="uaccess"' \
  | sudo tee /etc/udev/rules.d/61-ddcutil-amd-apu.rules
sudo udevadm control --reload && sudo udevadm trigger --subsystem-match=i2c-dev

getfacl -p /dev/i2c-21 | grep user:     # user:<you>:rw-
```

The I²C bus numbers also changed across the reboot. Before it, the portrait TCL was on `i2c-23`; after it, it was on `i2c-21`. A saved `ddcutil --bus N` command can then change the wrong monitor. Select by manufacturer or model instead:

```bash
ddcutil setvcp 10 70 --mfg TCL
```

## 4. One monitor still missing: concurrent DDC reads through a dock

With permissions fixed, the extension showed two of the three external monitors. Which monitor was missing changed from one run to the next. The logs:

```
Error detecting VCP version using VCP feature xDF: Error_Info[DDCRC_DISCONNECTED ...]
No monitor detected on bus /dev/i2c-22
```

The root cause is concurrency. Sequential reads always succeed. Overlapping reads through the dock fail:

```
$ for i in 1 2 3; do ddcutil getvcp 10 --bus 21 --brief; done
VCP 10 C 23 100
VCP 10 C 23 100
VCP 10 C 23 100

$ ddcutil getvcp 10 --bus 21 --brief & ddcutil getvcp 10 --bus 22 --brief & \
  ddcutil getvcp 10 --bus 21 --brief & wait
No monitor detected on bus /dev/i2c-21
VCP 10 C 23 100
VCP 10 C 27 100
```

The extension probes the buses in parallel. I had already patched it to probe one bus at a time, but it still started the next probe while the previous monitor's VCP reads were running. The current local patch waits 1.5 s between monitors and retries a "No monitor detected" result up to three times with the same delay:

```javascript
} else if (retried < 3 && response.indexOf('No monitor detected') !== -1) {
    GLib.timeout_add(GLib.PRIORITY_DEFAULT, 1500, () => {
        probeBus(index, retried + 1);
        return GLib.SOURCE_REMOVE;
    });
    return;
}
GLib.timeout_add(GLib.PRIORITY_DEFAULT, 1500, () => {
    probeBus(index + 1, 0);
    return GLib.SOURCE_REMOVE;
});
```

A timed delay only hides the race. The real fix is one queue for *every* `ddcutil` call the extension makes, so that no two calls ever overlap. That is the next step, and it is a better upstream patch than a delay.

GNOME Shell on Wayland caches extension modules. Turning an extension off and on does not load the edited file; you must log out and in again.

## Checklist for a multi-monitor GNOME Wayland desk

| Problem | Where the state lives | Tool |
|---|---|---|
| Layout, rotation, scale | `~/.config/monitors.xml`, one entry per connector set | Settings, then `gnome-randr` or Mutter D-Bus when Settings refuses |
| Xorg vs Wayland differences | Different connector names, so different entries | Fix each session type separately |
| DDC/CI permission | udev `uaccess` rule, by PCI class | `getfacl /dev/i2c-N`, check the GPU's `class` |
| DDC bus numbers | Change across reboots | `ddcutil --mfg`/`--model`, not `--bus` |
| Flaky DDC through a dock | Concurrent I²C transactions | Serialize all `ddcutil` calls |

## Source

- [Mutter DisplayConfig D-Bus interface](https://gitlab.gnome.org/GNOME/mutter/-/blob/main/data/dbus-interfaces/org.gnome.Mutter.DisplayConfig.xml)
- [gnome-randr (Rust)](https://github.com/maxwellainatchi/gnome-randr-rust)
- [ddcutil](https://github.com/rockowitz/ddcutil) and its [udev rule](https://github.com/rockowitz/ddcutil/blob/2.0.0-release/data/usr/lib/udev/rules.d/60-ddcutil-i2c.rules)
- [Brightness control using ddcutil (GNOME extension)](https://github.com/daitj/gnome-display-brightness-ddcutil)
- [autorandr](https://github.com/phillipberndt/autorandr)
- Earlier in this series: {% post_url 2026-09-20-why-electron-break-timers-fail-on-wayland %}
