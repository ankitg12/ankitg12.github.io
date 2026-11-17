---
layout: post
title: "Two Keyboards, One Dictation Toggle"
date: 2026-11-17
categories: linux voice debugging
series: "Moving to Linux"
---

My small keyboard had a right Control key but no numpad. My dictation shortcut was numpad 0/Insert. I needed both shortcuts: right Control on the small keyboard and the existing key when I switched keyboards.

[Voxtype](https://github.com/peteonrails/voxtype) was already running in toggle mode: press once to record, press again to transcribe. Its configuration named one key, `EVTEST_82`. In the [Linux input subsystem](https://www.kernel.org/doc/html/latest/input/input.html), that is the kernel event `KEY_KP0`; the Insert legend on a numpad key does not necessarily mean a dedicated `KEY_INSERT` event.

The working section in `~/.config/voxtype/config.toml` is:

```toml
[hotkey]
enabled = true
key = "EVTEST_82"
mode = "toggle"
```

The key name can be checked without guessing from the label. [evtest](https://gitlab.freedesktop.org/libevdev/evtest) prints the input events from a selected device:

```sh
evtest /dev/input/eventN
```

Replace `eventN` with the keyboard's device and press the key. Access to the input device is required.

## The first change broke the other keyboard

Voxtype accepts `RIGHTCTRL` as a hotkey. My agent changed the setting and restarted the service. The log showed that Voxtype was listening for `KEY_RIGHTCTRL`.

That proved the new setting loaded. It did not prove the original key still worked: the change had **replaced** it. I tried the old key, found that it no longer worked, and clarified that I needed both.

The agent restored `EVTEST_82`. Then it tried a second approach: use [input-remapper](https://github.com/sezanzeb/input-remapper) to translate right Control into `KEY_KP0`. The keyboard already had a preset for turning its volume wheel into scrolling, so the agent added the new mapping to that preset.

I stopped it there. A dictation shortcut does not belong in a wheel-scroll configuration, even if that configuration happens to own the keyboard. The agent removed the mapping and reloaded the original preset.

## A separate listener

The final setup leaves Voxtype's original binding in place and adds a separate listener using [Triggerhappy](https://github.com/wertarbyte/triggerhappy), a Linux input-event daemon. Its trigger file contains a single binding; the fields are separated by tabs:

```text
KEY_RIGHTCTRL	1	/home/<user>/.local/bin/voxtype record toggle
```

The `1` is the key-press event. The command asks the existing Voxtype daemon to toggle recording; it does not start a second speech engine. Replace `<user>` with the actual home-directory name. Triggerhappy was installed locally from the distribution package, without changing the system installation.

A [systemd user service](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html) runs the listener. The following is the service with the home path expressed as `%h`, the systemd specifier for the user's home directory:

```ini
[Unit]
Description=Voxtype right Control hotkey
PartOf=graphical-session.target
After=graphical-session.target voxtype.service

[Service]
ExecStart=%h/.local/opt/triggerhappy/usr/sbin/thd --triggers %h/.config/voxtype/right-control.conf --deviceglob /dev/input/event*
Restart=always
RestartSec=3

[Install]
WantedBy=default.target
```

Save it as `~/.config/systemd/user/voxtype-right-control.service`, then enable it:

```sh
systemctl --user daemon-reload
systemctl --user enable --now voxtype-right-control.service
```

The binary and trigger paths must match the local installation; a distribution-wide installation can use a different path. The listener needs permission to read the input devices.

| Key | Route |
|---|---|
| Numpad 0/Insert | Voxtype listens for `KEY_KP0` directly |
| Right Control | Triggerhappy receives `KEY_RIGHTCTRL` and runs `voxtype record toggle` |

There was one more input-layer detail. The wheel preset grabs the physical keyboard and forwards its other events through a virtual input device. The listener had that forwarded device open as well as the physical device. The agent also checked whether a desktop shortcut could use right Control on its own: the local [GTK](https://www.gtk.org/) accelerator check marked `Control_R` as invalid. That made a separate input-event listener a more suitable route for this key.

## What we tested

A synthetic `KEY_RIGHTCTRL` event took Voxtype from `idle` to `recording`. The agent cancelled that recording and confirmed the state returned to `idle`. That established the listener-to-Voxtype path, not the physical keyboard path.

The service and Voxtype state can be inspected separately:

```sh
systemctl --user is-active voxtype voxtype-right-control
voxtype status
journalctl --user -u voxtype-right-control --since '5 minutes ago'
```

An `active` service is only a readiness check. For an end-to-end test, press the physical key, check for `recording`, then press it again to stop. `voxtype record cancel` discards a test recording without transcribing it.

I then tested both physical keys. Right Control worked, and numpad 0/Insert worked too. This post itself began as dictation using right Control.

The hotplug case remains open. Triggerhappy opens the matching input devices when it starts; a keyboard reconnected later may need the listener restarted. The agent tried an automatic refresh on input-device changes, but the test did not pass, so it removed that attempt. I have not verified a reconnect.

## The lesson

A running service and a correct-looking log line are not the same as a working workflow. The first change satisfied the new key and silently removed the old one. The wheel preset then offered a convenient place to put a mapping, but convenience did not make it the right owner.

The useful test was the one that matched the request: try **both** keys, on the keyboards they are for.

## Source

- [Voxtype configuration and external recording controls](https://github.com/peteonrails/voxtype/blob/main/docs/CONFIGURATION.md)
- [Triggerhappy documentation](https://github.com/wertarbyte/triggerhappy)
- [input-remapper](https://github.com/sezanzeb/input-remapper)
- [Linux input event codes](https://www.kernel.org/doc/html/latest/input/event-codes.html)
