---
layout: post
title: "Put the indicator where the eye already is: a caret-anchored dictation OSD on GNOME"
date: 2026-10-07 17:00:00 +0530
categories: linux wayland gnome
---

My dictation tool showed its "recording" waveform on whichever of my four monitors the window manager chose, and sometimes the transcript was typed into a different window from the one I was looking at. This post records how I fixed both on GNOME Shell 46, the dead ends on the way, and a checklist so the next Shell extension costs an afternoon less.

The tool is [Voxtype](https://github.com/peteonrails/voxtype): push-to-talk speech-to-text that types the result with `ydotool`. The same pain is reported upstream in [#801](https://github.com/peteonrails/voxtype/issues/801) (focus restore, "inactive on GNOME"). Earlier posts on the same setup: [the speech recognition rabbit hole]({% post_url 2026-10-02-the-local-speech-recognition-rabbit-hole %}) and [the udev race that silenced the hotkey]({% post_url 2026-10-06-ten-milliseconds-silent-hotkey-udev-race %}).

## The idea is thirty years old

Mark Weiser and John Seely Brown wrote [*Designing Calm Technology*](https://calmtech.com/papers/designing-calm-technology.html) at Xerox PARC in December 1995. Their point: good technology "engages both the *center* and the *periphery* of our attention, and in fact moves back and forth between the two." A recording indicator must answer two questions at the moment I speak: *am I being heard?* and *where will the words go?* Both questions are about the caret, so the indicator belongs at the caret, and because it sits at the centre of attention, it must be small.

Two more precedents shaped the design:

| Precedent | What it says here |
|---|---|
| [Fitts's law](https://en.wikipedia.org/wiki/Fitts%27s_law) (Paul Fitts, 1954) | Cost grows with distance to the target. An indicator on another monitor costs a head turn every time you dictate. |
| X11R5 XIM, the [X Input Method](https://www.x.org/releases/X11R7.7/doc/libX11/XIM/xim.html) ([released 5 September 1991](https://en.wikipedia.org/wiki/X_Window_System)) | Built so CJK input methods could draw a candidate window *at the spot where the next character goes*. The caret position we need was already flowing through the desktop, for a different reason, since 1991. |

## Trajectory

### 1. Why the OSD went to a random monitor

Voxtype's GTK4 OSD ran under XWayland (`GDK_BACKEND=x11`, needed on this setup for keep-above). Under XWayland, Mutter places the window, so Voxtype's `[osd] position` setting has no effect.

### 2. First fix: anchor to the target window

- A `pre_recording_command` script records the focused window at record start. GNOME does not expose this to other processes, so I used the small `focused-window-dbus` extension, which returns the window id and geometry over D-Bus. The script skips the OSD's own window.
- The OSD wrapper polls that file every 0.25 s and moves the OSD with `xdotool windowmove` to the bottom-centre of the target window.

This works in every app, today, and it remains the fallback. But the bottom of a 2160 px window is still far from the caret.

### 3. Looking for prior work

- Three existing Voxtype GNOME extensions: one uses the monitor under the pointer, one uses the primary monitor, one shows a panel icon. None follows the caret, and none supports Shell 46.
- One open plan ([Olbrasoft/LinuxDesktop#79](https://github.com/Olbrasoft/LinuxDesktop/issues/79)) gets the caret from AT-SPI, the accessibility bus. It must walk the accessibility tree on every query (with a 250 ms timeout), and terminals such as WezTerm do not expose an AT-SPI text interface.

### 4. Where the caret actually lives

I first planned a custom IBus engine, because IBus receives the caret from every app. Reading the GNOME Shell source showed a shorter path: the Shell is already the IBus panel. [`js/misc/ibusManager.js`](https://gitlab.gnome.org/GNOME/gnome-shell/-/blob/46.0/js/misc/ibusManager.js) emits a `set-cursor-location` signal with the caret rectangle in screen coordinates. Apps reach it through IBus from three paths:

- X11 apps through XIM (`ibus-x11`)
- GTK and Qt apps through their IM modules
- Wayland apps through `text-input-v3`, bridged by the Shell

Dead ends worth recording:

- `Main.inputMethod._cursorRect` looks right but is updated only for Wayland text-input clients. My terminal is an X11 client, so it stayed empty.
- `org.gnome.Shell.Eval` over D-Bus returns `(false, '')` unless unsafe mode is on. Probing needs Looking Glass (Alt+F2, `lg`).

The Looking Glass probe, one line:

```js
const m = Main.panel.statusArea.keyboard._inputSourceManager._ibusManager; const id = m.connect('set-cursor-location', (_, r) => log('CARET ' + JSON.stringify(r) + ' ' + global.display.focus_window?.get_wm_class())); imports.gi.GLib.timeout_add(0, 20000, () => { m.disconnect(id); log('CARET-END'); return false; });
```

### 5. The probe dictated into itself

While Looking Glass was open, I dictated a question, and the text went *into Looking Glass*. GNOME still reported my terminal as the focused window. This is a second class of the wrong-window bug: Shell chrome (Looking Glass, overview search, the run dialog) takes keyboard focus without becoming a window, so any tool that asks "which window has focus?" cannot see it. The fix had to run inside the Shell.

### 6. A shell-script probe, and why it was the wrong tool

To measure carets from apps other than the terminal without Looking Glass, I wrote `ibus-caret-probe`: `dbus-monitor` on the IBus bus, piped into awk. It took three attempts. Each failure came from turning typed D-Bus messages into text and parsing that text again, not from the problem itself:

| Symptom | Cause |
|---|---|
| Empty log | `mawk` block-buffers pipe input. Use `gawk`, or `mawk -W interactive`. |
| Wrong fields | Positional parsing of `dbus-monitor` text meant for humans. |
| Leftover processes and NUL bytes in the log | Killing the script left the pipeline stages running. Fix: `trap 'kill 0'`, run the pipeline in the background and `wait`, because bash defers traps while a foreground command runs. |

The lesson for next time: when the data is typed messages, use a language that keeps the types (Python with GObject introspection, or GJS inside the Shell). Use shell only when the data really is lines of text.

### 7. What the apps reported

| App | Caret (x,y w×h) | Path |
|---|---|---|
| WezTerm | `80,1852 0x0` | X11 XIM |
| VS Code | `2246,1669 0x48` | Electron |
| Edge address bar | `1370,99 2x41` | Chromium |
| GNOME Text Editor | `2120,2254 0x22` | GTK4 |

Filters the extension needs:

- Drop `0,0 0x0`. It means "no caret" and arrives at every focus change.
- Drop wide rectangles. VS Code sent `3128x48`, a selection range, not a caret.
- Drop carets outside the target window. Text Editor sent one at x=1092 while its window was still opening at x=1987.

### 8. The extension, and three bugs found by screenshots

The extension, [`voxtype-caret`](https://github.com/ankitg12/voxtype-caret) (GPL-2.0-or-later, install steps in its README), does three things:

1. It watches Voxtype's state file. At `recording` it saves `global.display.focus_window` as the target.
2. It moves the Voxtype waveform window next to the last caret with `MetaWindow.move_frame()`. A marker file in `$XDG_RUNTIME_DIR` tells the old wrapper to stop its own moves; disabling the extension deletes the marker and the old behaviour returns.
3. It exports `RestoreFocus()` on D-Bus. Voxtype's `pre_output_command` calls it just before typing. It closes Looking Glass or the overview if open, then activates the saved window.

```js
RestoreFocus() {
    const win = this._target;
    if (!win || !win.get_compositor_private())
        return false;
    if (Main.lookingGlass?.isOpen)
        Main.lookingGlass.close();
    if (Main.overview.visible)
        Main.overview.hide();
    if (global.display.focus_window !== win)
        win.activate(global.get_current_time());
    return global.display.focus_window === win;
}
```

Three bugs, each found from one screenshot:

| Bug | Cause | Fix |
|---|---|---|
| The indicator always fell back to the window bottom | I cleared the caret at record start. Apps send carets only on keystrokes, and nobody types while dictating. | Keep the last caret, and reject it if it is outside the target window. |
| The 400×48 waveform covered the line above the prompt | Correct code, wrong design: the centre of attention needs a small indicator. | `[osd] width_px = 160`, `height_px = 24`; place it to the right of the caret, on the same line (empty while dictating), else below, else above. |
| "Below the caret" landed two lines too low in the terminal | An XIM spot is not a box. The [Xlib manual](https://www.x.org/releases/current/doc/libX11/libX11/libX11.html) says of `XNSpotLocation`: "The y coordinate is the position of the baseline used by the current text line." | Convert a `0x0` spot to a cell box above the baseline. |

The third bug is the 1991 design surfacing in 2026: the spot was built for a candidate window that sits *under the baseline*, not for a box around the caret.

## Checklist for the next GNOME Shell extension

1. **Read the Shell source for your version first.** The data you want is often already a signal inside the Shell. Extract the JS from `libgnome-shell.so` resources or browse the tag on GitLab.
2. **Probe in Looking Glass before writing a file.** One `connect(...)` line with a `timeout_add` that disconnects answers "does this signal fire for app X?" in a minute.
3. **Do not dictate or type into the probe.** Shell chrome holds keyboard focus without being a window.
4. **Use a nested Shell for the edit loop.** On Wayland, the Shell cannot restart while you are logged in, and disabling then enabling an extension does not reload its modules. Each code change cost me a log-out. The [GJS guide](https://gjs.guide/extensions/development/debugging.html) gives the fix I should have used from the start: `dbus-run-session gnome-shell --nested --wayland` (`--devkit` on GNOME 49+).
5. **Export a `Status()` method on D-Bus.** One `gdbus call` then shows state, target, caret and placement, which made every screenshot diagnosable.
6. **Design the rollback first.** Keep the old mechanism working, and let the new one take over by a marker that disappears on `disable()`.
7. **Expect private internals.** `_inputSourceManager._ibusManager` is not public API. Pin `shell-version` and test each Shell upgrade.

## Limits

- Tested end to end in WezTerm. Edge, VS Code and Text Editor report usable carets (table above), but I did not test the final placement there.
- If the target window was closed, `RestoreFocus()` returns false and the text goes wherever focus is. Voxtype treats `pre_output_command` as best effort, so the hook cannot cancel typing. A full guard needs an upstream option such as "non-zero exit sends the text to the clipboard instead".
- The extension is published at <https://github.com/ankitg12/voxtype-caret>. It is tested only on GNOME Shell 46, it uses a private Shell internal, and the terminal cell size is an estimate (see its README). Shell 47+ ports are welcome.

## Source

- The extension: <https://github.com/ankitg12/voxtype-caret>

- Voxtype: <https://github.com/peteonrails/voxtype>, issue [#801](https://github.com/peteonrails/voxtype/issues/801)
- GNOME Shell 46 `ibusManager.js`: <https://gitlab.gnome.org/GNOME/gnome-shell/-/blob/46.0/js/misc/ibusManager.js>
- Weiser & Brown, *Designing Calm Technology*, Xerox PARC, 1995: <https://calmtech.com/papers/designing-calm-technology.html>
- The Input Method Protocol (XIM), X Consortium: <https://www.x.org/releases/X11R7.7/doc/libX11/XIM/xim.html>
- Xlib manual, `XNSpotLocation`: <https://www.x.org/releases/current/doc/libX11/libX11/libX11.html>
- Fitts's law: <https://en.wikipedia.org/wiki/Fitts%27s_law>
- GJS guide, debugging and nested Shell: <https://gjs.guide/extensions/development/debugging.html>
- AT-SPI caret plan: <https://github.com/Olbrasoft/LinuxDesktop/issues/79>
