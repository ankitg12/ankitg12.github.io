---
layout: post
title: "Don't Screenshot Yourself: Capturing Occluded Windows on Windows"
date: 2026-07-25 16:30:00 +0530
categories: ai agents productivity windows
series: "AI coding agent productivity"
---

My terminal coding agent has a `/screenshot` command. On my laptop it kept photographing *itself* — a full-frame grab of the one screen the terminal already owned. The fix turned into a small tour of the three layers of screen capture on Windows, and two gotchas that only show up when a program tries to look at the world around it.

## The recursive-mirror problem

The original command was one line of Python:

```python
from PIL import ImageGrab
ImageGrab.grab().save(out)
```

`ImageGrab.grab()` with no bounding box grabs the **primary display in full**. On a docked multi-monitor desk that's often fine — the agent lives on one monitor, the thing you want to show it is on another. But on a bare laptop there is exactly one screen, and the terminal running the agent *is* that screen. So `/screenshot` handed the model a picture of the model's own window. A snake eating its tail.

The real intent of "screenshot" here is almost never "the whole primary display." It's "capture *that* window over there" — the browser, the dialog, the app I just alt-tabbed away from. So the fix is to stop grabbing displays and start grabbing **a specific window** — including one that is hidden *behind* the terminal.

That "behind" is the crux. You cannot `BitBlt` pixels that aren't on screen. You need an API that reads a window's own backing store, not the framebuffer.

## The three layers of Windows capture

Windows has accumulated three distinct capture mechanisms, each at a different level of the graphics stack:

| API | Level | Captures occluded? | GPU/DWM content? | Good for |
|---|---|---|---|---|
| **GDI** `BitBlt` / `PrintWindow` | Old GDI | `PrintWindow`: yes (redraws window) | Partially (`PW_RENDERFULLCONTENT`) | One window, zero deps |
| **Windows.Graphics.Capture (WGC)** | DWM composition (Win10 1803+) | Yes | Yes (reads the DWM surface) | Modern window/monitor capture |
| **DXGI Desktop Duplication** | GPU framebuffer | No (it's the *screen*) | Yes | High-FPS full-screen/monitor |

DXGI Desktop Duplication is the fastest, but it duplicates a *display*, not a window — wrong tool for "the window behind my terminal." That leaves `PrintWindow` and WGC.

### Layer 1: PrintWindow (the classic GDI recipe)

`PrintWindow` asks a window to render itself into a device context you provide — even if it's occluded, because the window redraws on demand rather than being copied off the screen:

```python
from ctypes import windll
PW_RENDERFULLCONTENT = 2  # Win8.1+: include DWM/GPU-composited content
windll.user32.PrintWindow(hwnd, save_dc.GetSafeHdc(), PW_RENDERFULLCONTENT)
```

That `PW_RENDERFULLCONTENT` flag matters: without it, hardware-accelerated windows (Chromium-based browsers, anything drawing through the DWM) come back blank. With it, most normal app windows capture correctly. It's zero-dependency (just `pywin32` + `Pillow`) and it addresses the window by **`HWND`** — a stable kernel handle. Remember that; it saves us later.

Its weakness: some surfaces still return **solid black** — transparent/layered overlay windows (think a "take a break" reminder), exclusive-fullscreen content, and the desktop itself. `PrintWindow` is asking the window to paint via GDI, and some windows simply don't honor that path.

### Layer 2: WGC (the modern DWM-surface path)

Windows.Graphics.Capture reads the **composited DWM surface** for a window or monitor. That's a fundamentally more capable place to tap: it sees exactly what the compositor sees, occlusion and GPU rendering included. The Python binding [`windows-capture`](https://pypi.org/project/windows-capture/) wraps it:

```python
from windows_capture import WindowsCapture, Frame, InternalCaptureControl

cap = WindowsCapture(cursor_capture=False, draw_border=False, window_name=title)

@cap.event
def on_frame_arrived(frame: Frame, ctl: InternalCaptureControl):
    frame.save_as_image(out_path)
    ctl.stop()

cap.start()
```

Two things worth knowing:

- **It's a stream, not a snapshot.** WGC is built for continuous capture, so you start it, grab the first frame in the callback, and immediately stop. For a single screenshot, run the pump on a **daemon thread** and poll a flag — otherwise a window that never produces a frame hangs you forever.
- **The capture border.** WGC draws a colored "you are being recorded" rectangle around the target by default. `draw_border=False` suppresses it — but only on Windows 11 (build 21H2+). On older builds it's forced on and would land in your screenshot.

## Gotcha 1: title matching vs. handle matching

Here's the lesson that made the whole detour worth it.

I let `/screenshot self` capture the terminal itself (useful for debugging the agent). WGC matches windows **by exact title string**. But my terminal's title contains a live braille spinner:

```
[1/2] π ⠋ Improve token generation speed
      ⠙ ...
      ⠹ ...
```

Between the moment I *resolved* the title and the moment WGC *looked it up*, the spinner character had already ticked over. WGC threw `Failed to find a window with the name: …`. Every single time.

`PrintWindow`, meanwhile, doesn't care — it captures by **`HWND`**, a handle that stays valid no matter how much the title churns. So the two-engine design fell out naturally: **WGC primary, PrintWindow fallback**, and anything with dynamic chrome deterministically lands on the handle-based path.

> The general principle: **address resources by stable handles, not by mutable display strings.** Titles are for humans; handles are for machines. Any window with a clock, a spinner, or a progress percentage in its title will defeat name-based lookup.

To make the fallback snappy I also watch the WGC pump thread's liveness — if it dies (title not found), bail to `PrintWindow` immediately instead of waiting out the 8-second timeout. `self` capture went from ~8s to ~1.7s.

## Gotcha 2: your window titles are full of invisible Unicode

The command crashed in production with:

```
UnicodeEncodeError: 'charmap' codec can't encode character '\u200b' in position 63
```

`U+200B` is a **zero-width space**. A browser had quietly injected one into its window title (`Microsoft​ Edge` — there's an invisible character in that string). Windows Python defaults `stdout` to the legacy **cp1252** code page, which has no mapping for it, so simply `print()`-ing the captured window's title blew up — *after* the capture had already succeeded.

The fix is one line, and it belongs in every Windows CLI that ever echoes text it didn't author:

```python
import sys
sys.stdout.reconfigure(encoding="utf-8", errors="replace")
```

Window titles, filenames, clipboard contents — any string from the outside world can carry arbitrary Unicode. cp1252 stdout is a landmine; step off it early.

## The final shape

```
capture(title):
  if title is a real substring match  -> WGC by exact resolved title
                                          -> on failure, PrintWindow by HWND
  if title == "self"                  -> PrintWindow by HWND (title too volatile for WGC)
  if no title, multiple monitors      -> full primary-monitor grab (the old behavior, still fine)
  if no title, single screen          -> topmost non-terminal window (what's behind the terminal)
```

Plus a **black-frame guard**: after any capture, check the mean pixel brightness; if it's ~0, treat it as a failed grab and fall through to the next engine. That single heuristic turns "silently returned a black rectangle" into "tried the other API."

Three capture layers, two addressing models, one invisible character. Not bad for a command that just takes a picture.

## Source

- [windows-capture on PyPI](https://pypi.org/project/windows-capture/) — the WGC Python binding
- [windows-capture on GitHub](https://github.com/NiiightmareXD/windows-capture)
- [Windows.Graphics.Capture — screen capture (Microsoft Learn)](https://learn.microsoft.com/en-us/windows/uwp/audio-video-camera/screen-capture)
- [`PrintWindow` (Win32 API reference)](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-printwindow)
- [DXGI Desktop Duplication API](https://learn.microsoft.com/en-us/windows/win32/direct3ddxgi/desktop-dup-api)
