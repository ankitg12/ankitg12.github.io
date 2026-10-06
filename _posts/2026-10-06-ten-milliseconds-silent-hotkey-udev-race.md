---
layout: post
title: "Ten Milliseconds: How a udev Permission Race Silenced My Dictation Hotkey"
date: 2026-10-06
categories: linux wayland debugging
---

I pressed Numpad 0 to start dictating, and nothing happened. The Voxtype daemon was running and the config had not changed. Last night it worked.

This post traces the fault down to a 10 ms window between two processes. The fix is about twenty lines of user-space shell, and I filed the bug upstream.

## The setup

| Layer | Role |
|---|---|
| Physical USB keyboard | sends `KEY_KP0` (evdev code 82) |
| [input-remapper](https://github.com/sezanzeb/input-remapper) | **grabs** the keyboard exclusively and sends its events out again on a virtual uinput device called `input-remapper <name> forwarded` (I use it to [turn a volume knob into a scroll wheel]({% post_url 2026-09-18-porting-keyboard-wheel-scroll-to-linux-wayland %})) |
| [Voxtype](https://github.com/peteonrails/voxtype) | reads `/dev/input/event*` directly and toggles recording on `EVTEST_82` |

The important property: when input-remapper has grabbed a keyboard, **every key is visible only on the forwarded device**. If a listener misses that one node, it misses every key.

## Step 1: Is the daemon alive and configured?

```
$ systemctl --user status voxtype
     Active: active (running) since Mon 2026-10-05 10:04:21 IST; 23h ago
```

```toml
[hotkey]
key = "EVTEST_82"   # KEY_KP0
mode = "toggle"
```

The last recording was the night before. Today's log had only one useful burst, from when the dock reconnected:

```
09:28:24.189183 INFO Opened keyboard: "/dev/input/event259" ("input-remapper DaKai Keyboard forwarded")
09:28:24.189249 INFO Opened keyboard: "/dev/input/event31"  ("input-remapper HID 1bcf:08a0 Keyboard forwarded")
09:28:24.689164 INFO Devices updated: 8 keyboard(s) active
```

So Voxtype saw the hotplug and rescanned the devices. That looks healthy.

## Step 2: Does the key reach the kernel?

Don't guess here. Measure. This [python-evdev](https://python-evdev.readthedocs.io/) loop prints each key-down with the device it came from:

```python
import evdev, select
devs = [evdev.InputDevice(p) for p in evdev.list_devices()]
devs = [d for d in devs if evdev.ecodes.EV_KEY in d.capabilities()]
fds = {d.fd: d for d in devs}
while True:
    r, _, _ = select.select(fds, [], [])
    for fd in r:
        for e in fds[fd].read():
            if e.type == evdev.ecodes.EV_KEY and e.value == 1:
                print(fds[fd].path, fds[fd].name, e.code, evdev.ecodes.KEY.get(e.code))
```

```
/dev/input/event261 input-remapper DaKai forwarded 82 KEY_KP0
/dev/input/event261 input-remapper DaKai forwarded 69 KEY_NUMLOCK
/dev/input/event261 input-remapper DaKai forwarded 82 KEY_KP0
```

That rules out two theories. NumLock does not matter: the evdev code is 82 in both states. And the key does reach the kernel. But it arrives on **`event261`**, and Voxtype's log never opened `event261`.

## Step 3: Why was event261 skipped?

When the kernel creates an evdev node, devtmpfs makes it `root:root 0600`. Shortly after, udev applies the rules and changes it to `root:input 0660`. A user in the `input` group can open the node only *after* that second step. `stat` records both moments:

```
$ stat -c '%n %U:%G %a birth=%w change=%z' /dev/input/event261 /dev/input/event259 /dev/input/event31
/dev/input/event261 root:input 660 birth=09:28:23.857410 change=09:28:24.199434
/dev/input/event259 root:input 660 birth=09:28:23.856410 change=09:28:24.123428
/dev/input/event31  root:input 660 birth=09:28:23.756403 change=09:28:23.987419
```

Put the two clocks side by side:

| Time | Event |
|---|---|
| 24.123 | udev gives `event259` to group `input` |
| **~24.189** | Voxtype rescans `/dev/input`: it opens `event259` and `event31`, and the open of `event261` fails with `EACCES` |
| **24.199** | udev gives `event261` to group `input`, **10 ms too late** |

Then the source code ([`src/hotkey/evdev_listener.rs`](https://github.com/peteonrails/voxtype/blob/main/src/hotkey/evdev_listener.rs)) shows why the miss is permanent:

1. The inotify watch is `CREATE | DELETE`. A chmod/chgrp is `IN_ATTRIB`, so udev's permission change does not wake the listener.
2. `try_open_device` drops `PermissionDenied` without even a trace line.
3. The rescan is one pass after a 150 ms settle delay. There is no retry.

So a device that was not ready on that one pass stays invisible until the daemon restarts. A restart proved it:

```
$ systemctl --user restart voxtype
INFO Opened keyboard: "/dev/input/event261" ("input-remapper DaKai forwarded")
INFO Listening for KEY_KP0 (with modifiers: {}) on 9 device(s)
```

## The fix that needs root, and the one that does not

The usual answer is a udev rule that tells the systemd user manager to start a unit:

```
ACTION=="add", SUBSYSTEM=="input", KERNEL=="event*", ATTRS{name}=="input-remapper * forwarded", \
  TAG+="systemd", ENV{SYSTEMD_USER_WANTS}+="voxtype-rescan.service"
```

My sudoers policy does not let me write to `/etc/udev/rules.d`. But reading udev events needs no root. `udevadm monitor` listens on the udev netlink socket as any user:

```bash
#!/usr/bin/env bash
set -euo pipefail
action=""
udevadm monitor --udev --subsystem-match=input --property | while read -r line; do
  case "$line" in
    ACTION=*) action="${line#ACTION=}" ;;
    'NAME="input-remapper '*' forwarded"')
      [[ "$action" == add ]] && systemctl --user start --no-block voxtype-rescan.service ;;
  esac
done
```

The target unit is a oneshot that waits and then restarts the daemon:

```ini
[Service]
Type=oneshot
ExecStartPre=/bin/sleep 2
ExecStart=/usr/bin/systemctl --user try-restart voxtype.service
```

The oneshot also debounces the events. A replug creates several forwarded devices within about a second. While the oneshot is still activating, systemd folds each later `start` into the same job. The watcher runs as an enabled user service, `WantedBy=default.target`.

To test it without a cable, ask input-remapper to recreate its devices:

```
$ input-remapper-control --command stop-all && input-remapper-control --command autoload
forwarded device added: NAME="input-remapper HID 1bcf:08a0 Keyboard forwarded"
Starting voxtype-rescan.service ...
forwarded device added: NAME="input-remapper DaKai Keyboard forwarded"
forwarded device added: NAME="input-remapper DaKai forwarded"
forwarded device added: NAME="input-remapper Generic USB Audio Consumer Control forwarded"
forwarded device added: NAME="input-remapper DaKai Consumer Control forwarded"
Finished voxtype-rescan.service ...
INFO Opened keyboard: "/dev/input/event259" ("input-remapper DaKai forwarded")
```

Five adds caused one restart, and the forwarded keyboard opened again.

## The general lesson

**Node creation is not the same as node readiness.** Any program that watches `/dev` with inotify and acts on `IN_CREATE` is racing udev. The correct designs are:

| Approach | Notes |
|---|---|
| Listen to udev (libudev / netlink), not the filesystem | udev sends the event *after* rules run, so permissions are already set |
| Also watch `IN_ATTRIB` | a cheap patch for an inotify-based design |
| Retry `EACCES` paths with backoff | belt and braces |
| Never drop an error silently | one `debug!` line would have shown this at once |

The same race affects anything that opens hot-plugged devices: serial ports, hidraw, GPUs. The timestamp from `stat -c %z` on the device node is the cheapest way to prove it.

## Source

- Upstream issue: [peteonrails/voxtype#842](https://github.com/peteonrails/voxtype/issues/842)
- Workaround: the full watcher and unit are shown above, and the udev-rule variant is in the upstream issue
- [input-remapper](https://github.com/sezanzeb/input-remapper)
- [`udevadm(8)`](https://man7.org/linux/man-pages/man8/udevadm.8.html), [`inotify(7)`](https://man7.org/linux/man-pages/man7/inotify.7.html)
- Earlier in this saga: [The Local Speech Recognition Rabbit Hole]({% post_url 2026-10-02-the-local-speech-recognition-rabbit-hole %})
