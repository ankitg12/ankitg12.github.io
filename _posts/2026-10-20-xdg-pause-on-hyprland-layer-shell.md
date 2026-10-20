---
layout: post
title: "xdg-pause on Hyprland: Same Break Timer, Layer-Shell Backend"
date: 2026-10-20
categories: linux hyprland productivity
series: "Moving to Linux"
---

[`xdg-pause`](https://github.com/ankitg12/xdg-pause) moved from GNOME (lx01) to Hyprland (Omarchy) today. Same break semantics, different fullscreen mechanism — and the port deleted more code than it added.

## What didn't change

- **Mini breaks (30s) are unskippable.** No resume button is created for `mini` at all (`if self.break_type == "long"` gates the button, the reason prompt, and the `Esc` early-exit path). `on_key_press` swallows every keystroke; the overlay exits only when the countdown hits zero.
- **Long breaks (600s):** strict 30s lock, then `Esc`/resume opens the inline reason prompt (min 5 chars, `Enter` to confirm, second `Esc` cancels back to the break).
- **Same CLI, same config, same hooks:** `xdg-pause [mini|long|auto|SECONDS]`, `xdg-pause snooze [MINUTES|start|end|status]`, `~/.local/state/xdg-pause/xdg-pause.log`, `$XDG_RUNTIME_DIR/xdg-pause.active` marker, all five lifecycle hooks.

## What changed: Xwayland fullscreen → layer-shell surfaces

On GNOME/Mutter, native GTK3 Wayland ignores `fullscreen_on_monitor` and `keep_above`, so `xdg-pause` forced `GDK_BACKEND=x11` and fought Xwayland `XAUTHORITY` cookies from multiplexer shells. On Hyprland, the native path works — via `gtk-layer-shell`:

```python
GtkLayerShell.init_for_window(win)
GtkLayerShell.set_layer(win, GtkLayerShell.Layer.OVERLAY)
GtkLayerShell.set_monitor(win, mon)
for edge in (TOP, BOTTOM, LEFT, RIGHT):
    GtkLayerShell.set_anchor(win, edge, True)
GtkLayerShell.set_exclusive_zone(win, -1)   # full-output cover, bar included
GtkLayerShell.set_keyboard_mode(win, GtkLayerShell.KeyboardMode.EXCLUSIVE)
```

Backend selection is automatic (`XDG_CURRENT_DESKTOP == Hyprland`) with a `XDG_PAUSE_BACKEND=layer-shell` override for other layer-shell compositors. On the layer-shell path the Mutter workarounds are skipped entirely: no `GDK_BACKEND=x11` forcing, no `_ensure_xauthority()`, no GNOME overview suppressor, no switcher-keybinding suspend.

The pixel proof, from `hyprctl layers` during a live break:

```text
Layer level 3 (overlay):
    Layer ...: xywh: 0 0 1920 1080, a: 1, namespace: xdg-pause
Layer level 3 (overlay):
    Layer ...: xywh: 1920 0 1366 768, a: 1, namespace: xdg-pause
```

Both outputs covered edge-to-edge — including the 26px top bar. `exclusive_zone(-1)` takes the whole output, so the break is genuinely pitch black.

## One rename along the way

The repo's systemd units are `xdg-pause.{service,timer}`, but the snooze default still pointed at the old lx01 name `xdg-fixed-break.timer` (survived as long as `XDG_PAUSE_TIMER` was unset nowhere). Default is now `xdg-pause.timer`; an explicit `XDG_PAUSE_TIMER=xdg-fixed-break.timer` still works for the old box.

## Verified on this box

- 8s mini: both monitors covered, auto-exit, zero layer surfaces left (`hyprctl layers` clean).
- Long: 600s overlay up on both monitors, `SIGTERM` exits cleanly with surfaces removed.
- Snooze: `snooze 60` stops the timer + schedules `break-snooze-resume.timer`; `snooze end` restarts immediately.
- Full suite: 16/16 unit tests pass.

## Source

- Commits: `d6cf8a7` (layer-shell backend), `68b3f2b` (timer rename)
- Prior post: {% post_url 2026-10-16-closing-the-muscle-memory-loop-in-xdg-pause %} — reason prompt, snooze, and hooks design (unchanged here)
