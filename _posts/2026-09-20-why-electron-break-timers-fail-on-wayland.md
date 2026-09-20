---
layout: post
title: "Why Electron Break Timers Fail on Wayland (and How Native GTK Fixes It)"
date: 2026-09-20 16:35:00 +0530
categories: linux wayland productivity
series: "Moving to Linux"
---

After migrating my primary developer workstation from Windows to Linux, I ran into an unexpected breakdown in my deep-work discipline: during scheduled breaks, I could keep typing.

On Windows, break reminder tools like [Stretchly](https://github.com/hovancik/stretchly) enforce genuine screen disengagement. When a 30-second mini-break or 10-minute long-break triggers, a fullscreen overlay completely covers the display. Keystrokes cannot bypass it.

On Linux GNOME Wayland, the break triggered, the visual overlay appeared, and yet my keystrokes kept routing directly into the terminal underneath. Even worse, pressing `Alt+Tab` brought the terminal right back to the top of the window stack. The break tool had become completely decorative.

This post documents why Electron break overlays fail on Wayland, why the naive workaround—hardware display power management (DPMS)—wrecks multi-monitor workspace topologies, and how a 250-line native GTK3 tool ([`xdg-pause`](https://github.com/ankitg12/xdg-pause)) cleanly solves the problem.

---

## The Upstream Bug: Why Wayland Ignores Electron Overlays

To understand why typing bypassed the break, I dug into Stretchly's window creation code in `app/main.js`:

```javascript
// Upstream Stretchly: app/main.js
const windowOptions = {
  skipTaskbar: !showBreaksAsRegularWindows,
  focusable: showBreaksAsRegularWindows,
  alwaysOnTop: !showBreaksAsRegularWindows,
  // ...
}

if (settings.get('fullscreen') && process.platform !== 'darwin') {
  windowOptions.width = displayManager.getDisplayWidth(localDisplayId)
  windowOptions.height = displayManager.getDisplayHeight(localDisplayId)
  windowOptions.x = displayManager.getDisplayX(localDisplayId, 0, true)
  windowOptions.y = displayManager.getDisplayY(localDisplayId, 0, true)
}

// ...
if (showBreaksAsRegularWindows) {
  microbreakWinLocal.show()
} else {
  microbreakWinLocal.showInactive()
}
```

Notice the defaults when `showBreaksAsRegularWindows` is `false`:
1. `focusable` is set to `false`.
2. The window is displayed via `showInactive()`.
3. Width and height are set manually to match display bounds, but `setFullScreen(true)` or `setKiosk(true)` is never invoked on non-macOS platforms.

### The Architectural Divergence: Win32 vs. Wayland

| Dimension | Windows (Win32 DWM) | Linux (GNOME Wayland / Mutter) |
|---|---|---|
| **Always-On-Top Layer** | `HWND_TOPMOST` z-order band | Ignored / standard `xdg_toplevel` layer |
| **Window Placement** | Precise global virtual screen coordinates (`x, y`) | Compositor controls placement; client coordinates ignored |
| **Focus Handling** | Topmost window visually absorbs mouse clicks | `showInactive()` + `focusable: false` leaves keyboard focus on background terminal |
| **Alt+Tab Behavior** | Background windows focus underneath `HWND_TOPMOST` | Compositor raises selected window to the top of the stack |
| **Memory Footprint** | ~150–200 MB Electron runtime | ~300–350 MB across multiple helper processes |

On Windows, the Desktop Window Manager strictly segments windows into z-order bands. A window marked `HWND_TOPMOST` draws above non-topmost windows regardless of whether it holds active keyboard focus. Mouse clicks are intercepted by the topmost surface.

On Linux Wayland, the `xdg_shell` protocol deliberately prevents unprivileged client applications from positioning themselves at arbitrary screen coordinates, manipulating global z-order, or stealing input focus from active applications.

Because Stretchly explicitly declared `focusable: false` and called `showInactive()`, it told GNOME's compositor (Mutter): *"Do not route keyboard input to this window."* Consequently, WezTerm never lost focus, and typing continued uninterrupted.

Toggling `showBreaksAsRegularWindows: true` only made matters worse: Stretchly disabled `alwaysOnTop`, allowing a single `Alt+Tab` to bury the break window underneath the terminal.

---

## The Trap: Why DPMS Screen Blanking Destroys Multi-Monitor Layouts

My initial reaction was to bypass the window manager entirely and enforce breaks at the hardware level using GNOME's D-Bus display power management interface:

```bash
# Blank displays immediately
busctl --user set-property org.gnome.Mutter.DisplayConfig \
  /org/gnome/Mutter/DisplayConfig \
  org.gnome.Mutter.DisplayConfig PowerSaveMode i 1

# Restore displays after break
busctl --user set-property org.gnome.Mutter.DisplayConfig \
  /org/gnome/Mutter/DisplayConfig \
  org.gnome.Mutter.DisplayConfig PowerSaveMode i 0
```

This produced a pitch-black screen. But on a dual-monitor setup (built-in laptop display + vertical external monitor), it introduced a fatal flaw:

1. Setting `PowerSaveMode i 1` cuts the video signal to the external monitor.
2. The external monitor enters standby and drops its HDMI/DisplayPort handshake.
3. GNOME Mutter detects the dropped signal as a **physical monitor disconnect**.
4. Mutter immediately collapses all virtual workspaces and migrates every open window from the external monitor onto the primary laptop screen.
5. When `PowerSaveMode i 0` restores video 30 seconds later, the external monitor wakes up completely empty. Your carefully arranged development layout is scrambled.

Hardware display sleeping is designed for machine idling, not interactive interval pacing.

---

## The Solution: Native Multi-Monitor GTK Surfaces (`xdg-pause`)

The correct design pattern on Wayland is straightforward: **keep the display signals alive, enumerate every active monitor, spawn a dedicated native GTK3 fullscreen surface on each, and swallow all keyboard events.**

This became [`xdg-pause`](https://github.com/ankitg12/xdg-pause).

```
   Dual Monitors (Signals Active)
┌──────────────────┐  ┌──────────────────┐
│ Monitor 0 (GTK3) │  │ Monitor 1 (GTK3) │
│ fullscreen surface│  │ fullscreen surface│
│ [██████░░] 24s   │  │ [██████░░] 24s   │
└──────────────────┘  └──────────────────┘
         │                      │
         └──────────┬───────────┘
                    ▼
     `key-press-event` Consumer
     (Swallows all keystrokes)
                    ▼
        Active Terminal / Slack
       (Zero input leaks through)
```

### 1. Spanning Monitors Without Signal Interruptions

Instead of setting window coordinate offsets, `xdg-pause` enumerates monitors through GDK and tells the compositor to make each window fullscreen on its designated monitor:

```python
display = Gdk.Display.get_default()
screen = display.get_default_screen()

for mon_idx in range(display.get_n_monitors()):
    win = Gtk.Window(type=Gtk.WindowType.TOPLEVEL)
    win.set_keep_above(True)
    win.fullscreen_on_monitor(screen, mon_idx)
    win.show_all()
    win.present()
```

Because video sync never drops, the external display never negotiates a disconnect. Window layouts remain completely static.

### 2. The Wayland Input Isolation Odyssey: What Failed vs. What Worked

Ensuring that no keystrokes or window-switching chords can leak into background editors on Wayland turned out to be an instructive engineering journey. Unlike Win32, where global keyboard hooks (`WH_KEYBOARD_LL`) and `HWND_TOPMOST` give clients absolute control, Wayland's security architecture is explicitly designed so that client applications cannot trap the user.

Here is the empirical record of what failed and what actually worked:

#### The Failures (and Why They Failed)

1. **`win.set_modal(True)`**:
   - *Attempt*: Mark overlay surfaces as modal dialogs to command all events.
   - *Outcome*: In Mutter, setting modal on multiple toplevel surfaces degraded the primary monitor window from `GDK_WINDOW_STATE_FULLSCREEN` into a centered dialog box, breaking fullscreen coverage on Monitor 0.

2. **`gdk_seat_grab(seat, Gdk.SeatCapabilities.ALL)`**:
   - *Attempt*: Grab the hardware seat to route all keyboard and pointer input exclusively to the overlay.
   - *Outcome*: Wayland seat grabs only apply to client-side event dispatching. The compositor (Mutter) intercepts global compositor shortcuts (`Alt+Tab`, `Super`) *before* client event delivery, rendering the seat grab powerless against compositor-level switching.

3. **`zwp_keyboard_shortcuts_inhibit_v1` (GTK4)**:
   - *Attempt*: Use the official Wayland protocol designed for virtual machine viewers (Spice/Remmina) to inhibit compositor shortcuts.
   - *Outcome*: GNOME Shell enforces an interactive security dialog on every invocation: *"Allow inhibiting shortcuts... You can restore shortcuts by pressing &lt;Super&gt;Escape"*. If unhandled or timed out, GNOME Shell flags the application as unresponsive and triggers a system *"App not responding"* freeze dialog. A break timer firing every 10 minutes cannot impose interactive permission prompts.

4. **Multi-Window `on_focus_out -> win.present()` Feedback Loop**:
   - *Attempt*: Call `win.present()` whenever `focus-out-event` fires.
   - *Outcome*: In a dual-monitor setup with two distinct windows, the display server allows only one surface to hold input focus. When Window 0 gained focus, Window 1 emitted `focus-out` and called `present()`. That stole focus from Window 0, which emitted `focus-out` and called `present()`. This triggered an infinite 4,800 events/second CPU ping-pong loop between the monitors, saturating the event loop.

5. **The Mystery `Gdk-CRITICAL` Assertion**:
   - *Symptom*: Windows emitted `assertion 'GDK_IS_WAYLAND_WINDOW (window)' failed` on realization.
   - *Root Cause*: A GDB backtrace under `G_DEBUG=fatal-criticals` proved the caller was not GTK, but Ubuntu's legacy `libappmenu-gtk-module.so` injected via `GTK_MODULES` (written for Unity top-bar menus in 2011). It hooked window realization on surfaces that had no menu bars.
   - *Fix*: Stripping `appmenu` from `os.environ["GTK_MODULES"]` before GTK initialization eliminated the assertion at the root cause without hacky log filters.

#### The Solutions That Worked

1. **In-Process GSettings Switcher Suspension**:
   Instead of trying to override the compositor from the bottom up, `xdg-pause` temporarily suspends GNOME Shell's switcher keybindings (`switch-windows`, `switch-applications`, `switch-group`, `switch-panels`) via in-process `Gio.Settings` in &lt;1ms on break start:
   ```python
   def suspend_gnome_switchers(self):
       self.wm_settings = Gio.Settings.new("org.gnome.desktop.wm.keybindings")
       for key in SUSPEND_KEYBINDINGS:
           self.saved_keybindings[key] = self.wm_settings.get_strv(key)
           self.wm_settings.set_strv(key, [])
   ```
   Because the keybindings are emptied in GSettings, Mutter does not intercept `Alt+Tab` or `Super+Tab`. Keystrokes fall directly into the overlay surface, where `on_key_press` swallows them. On exit (guaranteed via `atexit` and `signal` handlers), the original keybindings are restored.

2. **Real-Time D-Bus Overview Suppression**:
   Tapping `Super` alone toggles the GNOME Shell Activities Overview (`OverviewActive`). `xdg-pause` subscribes to `org.gnome.Shell`'s `PropertiesChanged` signal and checks a 100ms polling guard. The moment `OverviewActive` becomes `true`, it issues an instantaneous D-Bus `Set` call returning `OverviewActive` to `false` and re-asserts the overlay.

3. **Top-Right Live Clock Overlay**:
   A break overlay should orient the user in time. Using `Gtk.Overlay`, `xdg-pause` positions an unobtrusive date and live digital clock in the top-right corner of each display, styled with subtle contrast (`#777777` on `#000000`).

### 3. Decoupled Localization via `LocaleAdapter`

Rather than littering UI code with procedural language conditionals (`if locale == 'hi' ...`), translations and format templates are completely decoupled into `~/.config/xdg-pause/config.json`:

```json
{
  "language": "hi",
  "locales": {
    "hi": {
      "seconds_remaining": "{s} सेकंड शेष है",
      "minutes_remaining": "{m} मिनट शेष है",
      "minutes_seconds_remaining": "{m} मिनट {s} सेकंड शेष है",
      "resume_button": "वापस जाएं (Esc)"
    },
    "en": {
      "seconds_remaining": "{s} second{s_plural} remaining",
      "minutes_remaining": "{m} minute{m_plural} remaining",
      "minutes_seconds_remaining": "{m}m {s}s remaining",
      "resume_button": "Resume Work (Esc)"
    }
  }
}
```

The application logic formats durations purely through parameter substitution (`{s}`, `{m}`, `{s_plural}`), making it trivial to support any language without modifying code.

---

## Comparison Summary

| Metric | Stretchly (Electron) | DPMS Blanking | `xdg-pause` (Native GTK3) |
|---|---|---|---|
| **Wayland Focus Isolation** | ❌ Fails (keystrokes leak) | N/A (displays off) | ✅ 100% absorbed |
| **Multi-Monitor Stability** | ⚠️ Coordinate offsets ignored | ❌ Scrambles window layout | ✅ Perfect layout preservation |
| **Visual Aesthetics** | Minimal countdown | Pitch black | Pitch black + depleting progress bar |
| **Memory Footprint** | ~300–350 MB RAM | 0 MB (D-Bus call) | ~35 MB (exits on break finish) |
| **System Overhead** | Persistent background daemon | Persistent timer | Oneshot process via `systemd.timer` |

---

## Source

- Repository: [`ankitg12/xdg-pause`](https://github.com/ankitg12/xdg-pause)
- License: MIT
