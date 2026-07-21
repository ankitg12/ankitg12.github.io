---
layout: post
title: "Placing windows by title on Windows: nircmd lands them wrong, and why the fix is DPI awareness"
date: 2026-07-21
categories: windows tools productivity
---

Every time I relaunch a set of terminal windows, they cascade into a stack in the middle of the primary monitor, and I rearrange them by hand. Three SSH consoles I use together — a server, a client, and a host box — should snap to a fixed layout on one command. This is a solved problem on Linux (any tiling WM) but on Windows the obvious tools quietly do the wrong thing when a 4K monitor is scaled, and the failure looks like the tool working.

Here's the whole rabbit hole: the tool that *looks* right (`nircmd`) places windows in the wrong coordinate space on a scaled display, the fix is a DPI-awareness flag most examples omit, and there's a second gotcha where a cross-monitor move rescales the window mid-flight.

## First attempt: nircmd, the boring choice

[nircmd](https://www.nirsoft.net/utils/nircmd.html) (NirSoft, ~2003) is the Lindy-effect answer for scripting Windows from the command line. It moves and sizes a window by title:

```
nircmd win setsize stitle "ssh-server" 0 0 957 597
```

I computed a two-up-plus-bottom layout from the monitor's work area and fired three of these. On the laptop panel (100% scale) it was perfect. On the external 4K at 150% scale, the windows flew *above* the visible area. But nircmd returned success for every call.

## Diagnosis: read back where the window actually landed

Rule I keep relearning: when a placement tool "succeeds" but the result is wrong, stop guessing and read back the actual geometry. `GetWindowRect` tells you exactly where the window is:

```python
import ctypes
from ctypes import wintypes
user32 = ctypes.windll.user32

def rect_of(hwnd):
    r = wintypes.RECT()
    user32.GetWindowRect(hwnd, ctypes.byref(r))
    return r.left, r.top, r.right - r.left, r.bottom - r.top
```

Requested top-left `(-1234, -2700)`; actual `(-1403, -3069)`. Off by 169px horizontally, 369px vertically — and the offset scaled with the monitor's 150%. That ratio is the tell: this is a DPI-scaling mismatch, not a bug in the coordinates I computed.

## Root cause: two processes, two coordinate spaces

Windows has per-process *DPI awareness*. What `GetMonitorInfo` reports for a monitor's rectangle depends on the calling process's awareness mode:

- **System-DPI-aware** (`SetProcessDPIAware()`, the one every old example calls) — you get *logical* coordinates, scaled by the primary monitor's factor. On a mixed-DPI setup these do not match physical pixels on the secondary monitors.
- **Per-monitor-aware v2** — you get true physical pixels for every monitor.

My script called `SetProcessDPIAware()` and computed logical coordinates. nircmd is a *separate process* with its *own* (different) awareness, so it interpreted my numbers in a different space. Cross-process coordinate hand-off across a DPI boundary is the trap. No amount of coordinate math on my side fixes it reliably, because the two processes never agree on what `(-1234, -2700)` means.

## Fix: do everything in one process, per-monitor-aware v2

The clean fix is to drop the external tool and place windows in-process with `SetWindowPos`, under the per-monitor-v2 context — so enumerate, place, and verify all share one physical-pixel space:

```python
import ctypes
from ctypes import wintypes
user32 = ctypes.windll.user32

# DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2 = -4
user32.SetProcessDpiAwarenessContext(ctypes.c_void_p(-4))

SWP_NOZORDER, SWP_NOACTIVATE = 0x0004, 0x0010

def find_hwnd(substr):
    out = []
    proc = ctypes.WINFUNCTYPE(ctypes.c_bool, wintypes.HWND, wintypes.LPARAM)
    def cb(hwnd, _):
        if not user32.IsWindowVisible(hwnd):
            return True
        n = user32.GetWindowTextLengthW(hwnd)
        if n:
            b = ctypes.create_unicode_buffer(n + 1)
            user32.GetWindowTextW(hwnd, b, n + 1)
            if substr.lower() in b.value.lower():
                out.append(hwnd)
        return True
    user32.EnumWindows(proc(cb), 0)
    return out[0] if out else None

def place(substr, x, y, w, h):
    hwnd = find_hwnd(substr)
    if not hwnd:
        return
    user32.ShowWindow(hwnd, 9)  # SW_RESTORE (un-maximize first)
    user32.SetWindowPos(hwnd, 0, x, y, w, h, SWP_NOZORDER | SWP_NOACTIVATE)
```

Requested `(-1234, -2700)` → actual `(-1234, -2700)`. Pixel-exact, because there is no longer a coordinate hand-off between two differently-aware processes.

Under v2, the 4K monitor also finally reports its *real* `3840x2160` from `GetMonitorInfo`. Under system-DPI-aware it had been reporting a scaled-down `2743x1543` — the same lie that had thrown off the layout math in the first place.

## Second gotcha: cross-monitor moves rescale mid-flight

Moving a window from a 100% monitor to a 150% monitor fires `WM_DPICHANGED`, and well-behaved apps *resize themselves* in response. So a single `SetWindowPos` that crosses a DPI boundary sets the position, then the window rescales out from under you — the read-back comes back the wrong size.

The fix is a self-correcting retry: after the move, verify, and if it's off, place once more. The second call has no DPI transition (the window is already on the target monitor), so it sticks:

```python
def place(substr, x, y, w, h):
    hwnd = find_hwnd(substr)
    if not hwnd:
        return
    user32.ShowWindow(hwnd, 9)
    r = wintypes.RECT()
    for _ in range(2):
        user32.SetWindowPos(hwnd, 0, x, y, w, h, SWP_NOZORDER | SWP_NOACTIVATE)
        user32.GetWindowRect(hwnd, ctypes.byref(r))
        if abs(r.left - x) <= 2 and abs(r.right - r.left - w) <= 2:
            break
```

## Bonus: matching a monitor by name

I wanted `--monitor acer` to target a specific physical display, not an index that reshuffles when I dock. The intuitive call returns nothing useful:

```python
# DeviceString here is "Generic PnP Monitor" — useless
user32.EnumDisplayDevicesW(adapter, 0, dd, 0)
```

The EDID vendor code lives in the *device interface path*, which you only get by passing `EDD_GET_DEVICE_INTERFACE_NAME` (flag `1`) as the last argument:

```python
import re
user32.EnumDisplayDevicesW(adapter, 0, dd, 1)   # flag 1, not 0
m = re.search(r"DISPLAY#([A-Z]{3})", dd.DeviceID or "")
code = m.group(1) if m else None   # e.g. "ACR", "DEL", "CMN"
```

Those three-letter codes are the [PNP vendor IDs](https://uefi.org/pnp_id_list) — `ACR` = Acer, `DEL` = Dell, `CMN` = Chi Mei / Innolux (common laptop panels). Map them to friendly names and you can select a monitor by manufacturer.

## Don't-reinvent-the-wheel check: why a script at all?

Before writing any of this I checked the established options, because a general tiling manager would beat a bespoke script if one fit:

| Option | Verdict for "snap N named windows on command" |
|---|---|
| [komorebi](https://github.com/LGUG2Z/komorebi) | Excellent tiling WM, but its license forbids commercial/work use without a paid subscription, and it shows a splash on MDM-enrolled machines. Great for a personal box; a non-starter on a managed work laptop. |
| [FancyZones](https://learn.microsoft.com/en-us/windows/powertoys/fancyzones) (PowerToys) | Free, MIT, but zone-based: you snap windows into *zones*, and it does not natively match a window by *title substring*. A [CLI landed recently](https://github.com/microsoft/PowerToys/issues/25196) but it's layout-management, not "place this named window here." |
| nircmd | The right shape (place-by-title CLI) but DPI-unaware — the entire failure above. |
| In-process `SetWindowPos` | ~60 lines, no daemon, no license, DPI-correct. Fits "3 known windows, on demand." |

For a handful of specific named windows, the script is the genuine structural fit. If I wanted *general* zone-snapping for every app, FancyZones would win. The point of the exercise wasn't to build — it was to confirm building was correct, and it was for this narrow case.

## Ergonomic aside

Since the script picks a monitor and a layout, it's worth encoding the ergonomics too: put the two *peer* windows (server and client, which you diff row-against-row) **side by side at eye level**, so comparison is a horizontal glance and your neck stays neutral. Stacking is better for a single long log, ideally on a portrait monitor. The eye-level display — not the laptop panel you look down at — is where the windows you watch for long stretches belong.

## Takeaways

- On multi-monitor Windows, **DPI awareness is a coordinate contract**. A window-placement tool that runs in a different process with different awareness will land windows in the wrong space, and report success doing it.
- `SetProcessDpiAwarenessContext(-4)` (per-monitor v2) + in-process `SetWindowPos` keeps enumerate/place/verify in one space.
- Cross-monitor moves fire `WM_DPICHANGED`; place twice.
- `EnumDisplayDevices` needs flag `1` to expose the EDID vendor code for name matching.
- Always read back geometry — a placement tool's exit code is not evidence the window went where you asked.

## Source

- [nircmd](https://www.nirsoft.net/utils/nircmd.html) — NirSoft command-line Windows utility
- [SetProcessDpiAwarenessContext](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setprocessdpiawarenesscontext) / [DPI_AWARENESS_CONTEXT](https://learn.microsoft.com/en-us/windows/win32/hidpi/dpi-awareness-context)
- [WM_DPICHANGED](https://learn.microsoft.com/en-us/windows/win32/hidpi/wm-dpichanged)
- [EnumDisplayDevices](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-enumdisplaydevicesw) (note the `EDD_GET_DEVICE_INTERFACE_NAME` flag)
- [PNP vendor ID list](https://uefi.org/pnp_id_list)
- [komorebi](https://github.com/LGUG2Z/komorebi) · [FancyZones](https://learn.microsoft.com/en-us/windows/powertoys/fancyzones)
