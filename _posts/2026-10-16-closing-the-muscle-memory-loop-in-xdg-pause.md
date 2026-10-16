---
layout: post
title: "Closing the Muscle-Memory Loop in a Break Timer: Snooze, Conscious Early Exit, and Lifecycle Hooks"
date: 2026-10-16 20:50:00 +0530
categories: linux wayland productivity
series: "Moving to Linux"
---

A break timer that is impossible to bypass eventually gets killed during a live incident; a break timer with a single-key escape eventually gets dismissed by muscle memory before your conscious brain registers the screen went black.

A month after building [`xdg-pause`](https://github.com/ankitg12/xdg-pause) ([Why Electron Break Timers Fail on Wayland]({% post_url 2026-09-20-why-electron-break-timers-fail-on-wayland %})), four day-to-day failure modes showed up in real use:

1. **The 30-second `Esc` reflex:** Long breaks (600 seconds) lock input for the first 30 seconds (`strict_interval_seconds: 30`) and then unlock `Esc` for emergencies. Within two weeks, my left hand learned to wait 30 seconds and tap `Esc` out of habit.
2. **Meeting snooze drift:** Pausing the `systemd --user` timer before a 90-minute meeting meant remembering to run `systemctl --user start xdg-fixed-break.timer` afterward. Forget once, and the rest of the day has no breaks.
3. **Silent accountability gaps:** When I skipped a 10-minute break early, nothing recorded *why* in my daily work log.
4. **The Xwayland `XAUTHORITY` trap in long-lived shells:** Running `xdg-pause` from a terminal multiplexer pane printed `Authorization required, but no authorization protocol specified` twice and returned `Gdk.Display.get_default() == None`.

Here is how each was solved inside `xdg-pause`.

---

## 1. Deliberate Friction: Inline Reason Prompt on Early Exit

On a 10-minute break, the emergency exit should exist, but it must require **System 2 (conscious) thought** instead of a single motor reflex.

When `require_early_exit_reason` is enabled (the default), pressing `Esc`, `Space`, `Enter`, or clicking the resume button after the 30-second strict window no longer destroys the overlay. Instead, it hides the button and reveals a synchronized `Gtk.Entry` across every monitor:

```python
def on_key_press(self, win, event):
    elapsed = time.time() - self.start_time
    if self.break_type == "long" and elapsed >= self.strict_interval:
        if self.prompting_reason:
            if event.keyval == Gdk.KEY_Escape:
                self.cancel_early_exit_prompt()
                return True
            # Route normal typing into the focused Gtk.Entry
            return False
        if event.keyval in (Gdk.KEY_Escape, Gdk.KEY_Return, Gdk.KEY_space):
            self.request_early_exit()
            return True
    return True
```

Key mechanics:
- **Selective key passthrough:** While the overlay is in normal countdown mode, `on_key_press` returns `True` for every keystroke to swallow stray typing. Once `prompting_reason` is active, `on_key_press` returns `False` for non-`Escape` keys so `Gtk.Entry` receives the text.
- **Multi-monitor mirroring:** Because a multi-monitor setup creates one fullscreen `Gtk.Window` per monitor, the `changed` signal mirrors the typed characters across all monitors' `Gtk.Entry` widgets so you can see what you are typing regardless of which screen you are looking at.
- **Minimum length gate (`min_reason_chars: 5`):** Pressing `Enter` on an empty string or a 1-character mash highlights the border in red (`entry.reason-entry.invalid`) and keeps the break active.
- **Second `Esc` cancels the prompt:** If tapping `Esc` the first time made you realize you do not actually have an emergency, pressing `Esc` again closes the input box and returns to the break countdown.

---

## 2. Built-in Timer Snooze (`xdg-pause snooze`)

Instead of managing `systemctl --user` units by hand during meetings, `xdg-pause` now manages its own timer lifecycle via transient systemd timers (`systemd-run --user --on-active`):

```bash
# Snooze breaks for 90 minutes (stops timer, kills any active overlay,
# and schedules break-snooze-resume.timer to restart the timer in 90m)
xdg-pause snooze 90

# Snooze indefinitely until explicitly resumed
xdg-pause snooze start

# Resume break timer immediately and cancel any pending auto-resume unit
xdg-pause snooze end

# Inspect active timer and snooze countdown
xdg-pause snooze status
```

Under the hood, `xdg-pause snooze 90` stops `xdg-fixed-break.timer` (configurable via `XDG_PAUSE_TIMER`), sends `SIGTERM` to any running `xdg-pause` overlay process so its `atexit` handler cleanly restores GNOME Shell's `Alt+Tab` keybindings, and schedules a transient user timer:

```python
subprocess.run(
    [
        "systemd-run",
        "--user",
        "--quiet",
        "--unit=break-snooze-resume",
        f"--on-active={mins}m",
        "--description=Resume xdg-pause breaks after snooze",
        "systemctl",
        "--user",
        "start",
        SNOOZE_TIMER,
    ],
    check=True,
)
```

Because the resume schedule lives inside `systemd --user`, it survives terminal exits and shows up directly in `systemctl --user list-timers`.

---

## 3. Lifecycle Hooks (`hooks` and `hooks/<event>.d/`)

Logging a skipped break to a personal journal (such as Logseq) is workflow-specific and does not belong hardcoded inside an upstream GTK utility. Instead, `xdg-pause` exposes five lifecycle events:

| Event | When it fires | Key Environment Variables |
|---|---|---|
| `break_start` | Overlay windows map on screen | `XDG_PAUSE_MODE`, `XDG_PAUSE_DURATION` |
| `break_complete` | Countdown reaches `0` naturally | `XDG_PAUSE_ELAPSED`, `XDG_PAUSE_DURATION` |
| `early_exit` | User submits early-exit reason | `XDG_PAUSE_REASON`, `XDG_PAUSE_ELAPSED`, `XDG_PAUSE_REMAINING` |
| `snooze_start` | `xdg-pause snooze` pauses timer | `XDG_PAUSE_DURATION` (snooze seconds, or `0` if indefinite) |
| `snooze_end` | `xdg-pause snooze end` resumes timer | `XDG_PAUSE_EVENT=snooze_end` |

Hooks can be configured either as inline shell commands in `~/.config/xdg-pause/config.json` or as executable scripts in `~/.config/xdg-pause/hooks/<event>.d/`:

```json
{
  "durations": {
    "mini_seconds": 30,
    "long_seconds": 600,
    "strict_interval_seconds": 30,
    "require_early_exit_reason": true,
    "min_reason_chars": 5
  },
  "hooks": {
    "break_start": null,
    "break_complete": "if [ \"$XDG_PAUSE_MODE\" = \"long\" ]; then lsq -a \"$(date -d \"-${XDG_PAUSE_DURATION} seconds\" +%H:%M) - $(date +%H:%M) [break] 10m long break completed\"; fi",
    "early_exit": "lsq -a \"$(date +%H:%M) [break-skip] Skipped ${XDG_PAUSE_MODE} break after ${XDG_PAUSE_ELAPSED}s/${XDG_PAUSE_DURATION}s — ${XDG_PAUSE_REASON}\"",
    "snooze_start": null,
    "snooze_end": null
  }
}
```

Every time a long break is ended early, the reason typed into the overlay is automatically appended to the day's journal alongside the elapsed and total seconds.

---

## 4. Bonus Bug: Why `GDK_BACKEND=x11` Failed in Multiplexed Shells

As documented in the [2026-09-28 update]({% post_url 2026-09-20-why-electron-break-timers-fail-on-wayland %}), `xdg-pause` forces `GDK_BACKEND=x11` before importing GTK so the overlay runs through Xwayland, which honors `fullscreen_on_monitor()` and `set_keep_above()` under GNOME Mutter.

However, invoking `xdg-pause -h` from a terminal multiplexer pane printed:

```text
Authorization required, but no authorization protocol specified
Authorization required, but no authorization protocol specified
```

And invoking `xdg-pause` without `-h` from that same shell crashed with `Gdk.Display.get_default() == None`.

### Root Cause
1. PyGObject's `from gi.repository import Gtk` override calls `Gtk.init_check()` at **import time**—before `argparse` even parses `-h`.
2. Under GNOME Wayland ([Mutter MR !626](https://gitlab.gnome.org/GNOME/mutter/-/merge_requests/626)), Mutter starts Xwayland on demand with a randomized MIT-MAGIC-COOKIE-1 file at `$XDG_RUNTIME_DIR/.mutter-Xwaylandauth.XXXXXX` and exports `XAUTHORITY` to the `systemd --user` manager.
3. Shells spawned by a persistent multiplexer server or daemon that started before (or outside) that environment variable export have `DISPLAY=:0` and `WAYLAND_DISPLAY=wayland-0`, but no `XAUTHORITY`. Without `XAUTHORITY`, Xwayland rejects the X11 client connection twice.

### Fix
Before importing `gi.repository.Gtk`, `_ensure_xauthority()` checks whether `XAUTHORITY` is set; if missing, it queries `systemctl --user show-environment` and falls back to the newest `$XDG_RUNTIME_DIR/.mutter-Xwaylandauth.*` cookie on disk:

```python
def _ensure_xauthority():
    if os.environ.get("XAUTHORITY"):
        return
    try:
        env_out = subprocess.check_output(
            ["systemctl", "--user", "show-environment"],
            text=True,
            stderr=subprocess.DEVNULL,
        )
        for line in env_out.splitlines():
            if line.startswith("XAUTHORITY="):
                candidate = line.split("=", 1)[1].strip()
                if os.path.exists(candidate):
                    os.environ["XAUTHORITY"] = candidate
                    return
    except Exception:
        pass
    rt_dir = os.environ.get("XDG_RUNTIME_DIR", f"/run/user/{os.getuid()}")
    auths = sorted(
        glob.glob(os.path.join(rt_dir, ".mutter-Xwaylandauth.*")),
        key=os.path.getmtime,
    )
    if auths:
        os.environ["XAUTHORITY"] = auths[-1]
```

---

## Source

- Repository: [`ankitg12/xdg-pause`](https://github.com/ankitg12/xdg-pause) (commits [`5d39ffb`](https://github.com/ankitg12/xdg-pause/commit/5d39ffb), [`64d9e47`](https://github.com/ankitg12/xdg-pause/commit/64d9e47), [`097f67b`](https://github.com/ankitg12/xdg-pause/commit/097f67b))
- Prior post: [Why Electron Break Timers Fail on Wayland (and How Native GTK Fixes It)]({% post_url 2026-09-20-why-electron-break-timers-fail-on-wayland %})
