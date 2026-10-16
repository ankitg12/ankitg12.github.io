---
layout: post
title: "Authorization Required: Why Persistent Terminal Multiplexers Break Xwayland After a Compositor Restart"
date: 2026-10-16
categories: linux wayland debugging
series: "Moving to Linux"
---

Running `xdg-pause -h` from an interactive shell inside a persistent terminal multiplexer printed two lines to `stderr` before `argparse` even reached `--help`:

```text
$ xdg-pause -h
Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

usage: xdg-pause [-h] [-v] [-s STRICT] [mode] [extra ...]
```

Worse, invoking `xdg-pause` without `-h` from the same shell failed with `Gdk.Display.get_default() == None`, and every X11 client (`xdotool`, `xprop`, `xrandr`) failed the same way—while the systemd user timer (`xdg-fixed-break.timer`) continued to launch the break overlay every ten minutes without error.

## Why `-h` Touches the Display Server at All

`xdg-pause` is a GTK3 multi-monitor break overlay (`{% post_url 2026-09-20-why-electron-break-timers-fail-on-wayland %}`). Because native GTK3 on GNOME Wayland ignores `Gtk.Window.fullscreen_on_monitor()` and `keep_above()`, `xdg-pause` sets `GDK_BACKEND=x11` before importing GTK so the overlay renders through Xwayland.

PyGObject's `gi.overrides.Gtk` module calls `Gtk.init_check()` at `import` time:

```text
$ python3 -c 'import os, gi; os.environ["GDK_BACKEND"] = "x11"; gi.require_version("Gtk", "3.0"); from gi.repository import Gtk, Gdk; print("Display:", Gdk.Display.get_default())'
Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Display: None
```

Before `main()` parses a single command-line flag, `libX11` attempts to authenticate against `$DISPLAY` (`:0`, Xwayland) twice and fails.

## The Cookie That Expired Behind the Multiplexer's Back

On classic Xorg, the `MIT-MAGIC-COOKIE-1` token lives at a stable path (`~/.Xauthority`). On GNOME Wayland, Mutter generates a random per-session Xwayland auth file under `$XDG_RUNTIME_DIR`:

```text
/run/user/1000/.mutter-Xwaylandauth.0VDFX3
```

Compare the environment of the live GUI terminal (`wezterm-gui`) with the environment of the persistent multiplexer daemon (`tern` / `tmux`) that survived an earlier `gnome-shell` restart:

```text
# Live GUI process launched after the compositor restart:
$ tr '\0' '\n' < /proc/$(pgrep -o wezterm-gui)/environ | grep -E 'DISPLAY|WAYLAND|XAUTH'
DISPLAY=:0
XAUTHORITY=/run/user/1000/.mutter-Xwaylandauth.0VDFX3
WAYLAND_DISPLAY=wayland-0

# Persistent multiplexer daemon started the previous day:
$ tr '\0' '\n' < /proc/$(pgrep -o tern)/environ | grep -E 'DISPLAY|WAYLAND|XAUTH'
DISPLAY=:0
XAUTHORITY=/run/user/1000/.mutter-Xwaylandauth.GH2IX3
WAYLAND_DISPLAY=wayland-0
```

Look closely at `$XDG_RUNTIME_DIR`:

| File | Mtime | Status |
|---|---|---|
| `/run/user/1000/.mutter-Xwaylandauth.GH2IX3` | Yesterday 15:17 | Readable on disk, **expired cookie** |
| `/run/user/1000/.mutter-Xwaylandauth.0VDFX3` | Today 12:54 | Active Xwayland cookie in `systemd --user` |

When `gnome-shell` restarted at 12:54, Mutter spawned a new Xwayland instance, wrote a fresh cookie to `.mutter-Xwaylandauth.0VDFX3`, and updated the `systemd --user` environment (`systemctl --user show-environment`). That is why systemd user timers continued to work: systemd injected the new `XAUTHORITY` into every service unit it spawned.

Meanwhile, the persistent terminal multiplexer daemon kept the environment it inherited the previous day—pointing `XAUTHORITY` at `.mutter-Xwaylandauth.GH2IX3`. Because the old file still existed on disk, a naive `[ -r "$XAUTHORITY" ]` or `os.path.exists(os.environ["XAUTHORITY"])` check passed while `libX11` presented yesterdays cookie to todays Xwayland server.

## Fix 1: Refresh `XAUTHORITY` by Mtime in `~/.bashrc`

Checking whether `$XAUTHORITY` is non-empty or readable is not enough when the old `.mutter-Xwaylandauth.*` file remains in `$XDG_RUNTIME_DIR`. Every new shell inside `tmux`, `tern`, or `herdr` should bind `XAUTHORITY` to the *newest* Mutter Xwayland cookie before the interactive guard in `~/.bashrc`:

```bash
# Refresh XAUTHORITY to the newest Mutter Xwayland cookie so persistent
# multiplexers (tern/tmux) never pass an expired cookie after a compositor restart.
_latest_xauth="$(ls -t "${XDG_RUNTIME_DIR:-/run/user/$(id -u)}"/.mutter-Xwaylandauth.* 2>/dev/null | head -n1)"
[ -n "$_latest_xauth" ] && export XAUTHORITY="$_latest_xauth"
unset _latest_xauth
```

## Fix 2: Self-Healing Display Lookup in `xdg-pause`

For tools like `xdg-pause` that force `GDK_BACKEND=x11` on Wayland, resolving the newest `.mutter-Xwaylandauth.*` file before `import gi` makes the binary immune to stale caller environments (such as an existing shell pane that was already open before the compositor restarted):

```python
def _ensure_xauthority():
    """Always prefer the newest Mutter Xwayland auth cookie over a stale multiplexer XAUTHORITY."""
    rt_dir = os.environ.get("XDG_RUNTIME_DIR", f"/run/user/{os.getuid()}")
    auths = sorted(
        glob.glob(os.path.join(rt_dir, ".mutter-Xwaylandauth.*")),
        key=os.path.getmtime,
    )
    if auths:
        os.environ["XAUTHORITY"] = auths[-1]
```

Verifying against an explicitly poisoned environment variable:

```text
$ XAUTHORITY=/run/user/1000/.mutter-Xwaylandauth.GH2IX3 bash --rcfile ~/.bashrc -ic 'echo "XAUTHORITY=$XAUTHORITY"; xdotool getDisplayGeometry; xdg-pause -v'
XAUTHORITY=/run/user/1000/.mutter-Xwaylandauth.0VDFX3
1920 1200
xdg-pause 0.2.0
```

## Source

- [`xdg-pause`](https://github.com/ankitg12/xdg-pause) — commit [`3d5b445`](https://github.com/ankitg12/xdg-pause/commit/3d5b445)
- Prior posts in this thread: [Why Electron Break Timers Fail on Wayland]({% post_url 2026-09-20-why-electron-break-timers-fail-on-wayland %}) and [Closing the Muscle-Memory Loop in a Break Timer]({% post_url 2026-10-16-closing-the-muscle-memory-loop-in-xdg-pause %})
