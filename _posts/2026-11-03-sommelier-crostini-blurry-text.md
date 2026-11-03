layout: post
title: "My Terminal Went Hazy: Sommelier's Two Density Channels and Blurry Text in Crostini"
date: 2026-11-03 10:15:00 +0530
categories: chromebook linux wayland
series: "Chromebook"
---
Text in [Ghostty](https://ghostty.org/) looked soft — windowed and fullscreen alike. Not broken: hazy, like the whole terminal was rendered one pixel off. Every setting I checked held. The fix was one line: `font-size = 12` → `13`. This post is why that worked, and what [Sommelier](https://chromium.googlesource.com/chromiumos/platform2/+/main/vm_tools/sommelier/README.md) — the invisible compositor between your Linux apps and the Chromebook screen — does to your pixels.

Chromebook (Athlon Silver 3050U, 2 cores visible), Debian 12 `penguin` container, Ghostty 1.1.3 over native Wayland, on the built-in panel. Suspects, all cleared:

```text
$ cat ~/.config/fontconfig/fonts.conf     # grayscale + hintfull — held
$ gsettings get org.gnome.desktop.interface font-antialiasing   # 'grayscale' — held
$ tr '\0' '\n' < /proc/842/environ | grep -E "GDK_BACKEND|WAYLAND_DISPLAY"
WAYLAND_DISPLAY=wayland-0                 # native Wayland, no Xwayland fallback — held
```

No regression anywhere. So the haze wasn't a setting. It was the pipeline.

## What Sommelier is

Your container has no display. ChromeOS does. [Sommelier](https://chromium.googlesource.com/chromiumos/platform2/+/main/vm_tools/sommelier/README.md) is the proxy in between: in its own words, *"a Wayland compositor that delegates compositing to a 'host' compositor"*, built to run *"inside a tight jail or virtual machine"*. It lives in [`chromiumos/platform2/vm_tools/sommelier/`](https://chromium.googlesource.com/chromiumos/platform2/+/main/vm_tools/sommelier), and the [crosvm book](https://crosvm.dev/book/devices/wayland.html) describes it the same way: *"a proxy Wayland compositor"* forwarding guest protocol to the host.

The full path from keystroke to photon:

```text
guest app (Ghostty/GTK)
  → Sommelier (guest-side proxy compositor, in the container)
  → virtio-wayland (Wayland protocol over virtio, through the guest GPU device)
  → Exo (ChromeOS host compositor on the Aura Shell)
  → Ash / panel
```

Three links worth naming. The host side is [Exo](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/components/exo/README.md) — *"a display server on top of the Aura Shell"* whose listed clients include *"Crostini (Linux apps on Chrome OS)"*. The guest→host channel is [virtio-wayland](https://alyssa.is/using-virtio-wl/) — the Wayland protocol carried over virtio via the virtual GPU (`--virtgpu-channel`, `/dev/wl0`), not a socket forward. And there's no GPU behind any of it in the container: no `/dev/dri`, so [Mesa](https://docs.mesa3d.org/) falls back to [llvmpipe](https://docs.mesa3d.org/drivers/llvmpipe.html), its *"fastest software rasterizer"* — every frame rasterized on two shared host cores. (I measured that cost separately: one GUI app sat at [200%+ CPU on llvmpipe]({% post_url 2026-11-03-tern-200-percent-cpu-llvmpipe %}).)

## Two paths through the proxy: Wayland vs X11

| | Native Wayland | X11 via Xwayland |
|---|---|---|
| Scale handling | Client submits higher-resolution buffers for the scale factor | Compositor treats client as LoDPI, bitmap-upscales the output |
| Fractional scale | Re-renders sharp | Blurry by construction |
| ChromeOS stance | *"preferred"* — [Linux on ChromeOS FAQ](https://chromeos.dev/en/linux/linux-on-chromeos-faq) | *"you might run into some compatibility kinks"* |

The mechanism is generic [Wayland fractional-scaling](https://discourse.gnome.org/t/full-explanation-of-current-hidpi-fractional-and-integer-scaling-support-in-wayland/14225) behavior (explained by a Mutter dev): X11 clients are *"made to believe it's LoDPI"* and *"compositors will scale them up, and this is why they are blurry"*. ChromeOS's own FAQ says it plainly: *"If you're applying a system-wide zoom or otherwise changing the default display resolution, we attempt to scale the application output to match. This can lead to blurry results."* Lesson one: run native Wayland — I verified mine with `/proc/<pid>/environ` plus the `wayland-cursor` memfd, and had previously removed a stale `GDK_BACKEND=x11` launcher override that forced the worse path (that saga is [its own post]({% post_url 2026-10-20-ghostty-external-display-flicker %})). Note `wmctrl`/`xdotool` are blind through Sommelier (no EWMH window list) and `xclip`/`wl-paste` block forever — same proxy cause.

## Two density channels, one panel

Here is the thing that cracked it. My container isn't running one display pipeline — it's running two, side by side:

```text
$ systemctl --user show-environment | tr ' ' '\n' | grep -iE "display|density|cursor"
DISPLAY=:0
DISPLAY_LOW_DENSITY=:1
WAYLAND_DISPLAY=wayland-0
WAYLAND_DISPLAY_LOW_DENSITY=wayland-1
XCURSOR_SIZE=24
XCURSOR_SIZE_LOW_DENSITY=12
```

Every channel doubled: two X displays (`:0`, `:1`), two Wayland sockets (`wayland-0`, `wayland-1`), two cursor sizes (24 vs 12). The live process list shows why — four Sommelier instances, each pair split the same way:

```text
sommelier -X --x-display=0 ... --stable-scaling ...   # hi-density X
sommelier -X --x-display=1 ... --stable-scaling ...   # low-density X
sommelier --parent --socket=wayland-0 ... --stable-scaling ...   # hi-density Wayland
sommelier --parent --socket=wayland-1 ... --stable-scaling ...   # low-density Wayland
```

ChromeOS keeps a hi-density and a low-density channel so a window dragged between the built-in panel and an external monitor lands on the right raster without restarting the app. The `Xft.dpi` on my session reads 96 — the bucketed value, not the panel's physical DPI. And `xrandr` shows a single `XWAYLAND0` at 1366x768 right now: the external monitor from the [flicker post]({% post_url 2026-10-20-ghostty-external-display-flicker %}) is disconnected, but both channels are still up.

That's the haze mechanism. The app renders into one channel's density; the host compositor ([Exo](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/components/exo/README.md)) maps it onto the physical panel through a contents-scale multiplier — Sommelier's [`--scale=SCALE`](https://chromium.googlesource.com/chromiumos/platform2/+/main/vm_tools/sommelier/README.md) (*"contents scaling... applied to both native wayland clients and X11 clients"*, e.g. `--scale=0.5` renders at half density), with [`--dpi=`](https://chromium.googlesource.com/chromiumos/platform2/+/main/vm_tools/sommelier/README.md) bucket-mapping (exact 112 DPI → 96 given buckets 72/96/160/240). Any non-integer step in that mapping resamples the bitmap — interpolation, not alignment — and text, being all high-frequency edges, shows it first. When I scaled things down for the external monitor earlier, the low-density channel's factor stuck to this setup; size-12 glyphs sat exactly at the blur threshold. ChromeOS's own [FAQ](https://chromeos.dev/en/linux/linux-on-chromeos-faq) admits it: *"If you're applying a system-wide zoom or otherwise changing the default display resolution, we attempt to scale the application output to match. This can lead to blurry results."* Same complaint class upstream: [VS Code blurry on ChromeOS](https://github.com/Microsoft/vscode/issues/64070) (fixed by Display size 100%), [Ghostty blurry fonts under fractional scaling](https://github.com/ghostty-org/ghostty/issues/1938), [Crostini apps not scaling at 100%](https://issues.chromium.org/41462260), plus [chromium-os-discuss](https://groups.google.com/a/chromium.org/g/chromium-os-discuss/c/y7e0Ewuqleg) discussion.

## Subpixel rendering dies under resampling

The second half of the fix was already in place. [fontconfig](https://wiki.archlinux.org/title/Font_configuration) has three knobs that matter here:

| Key | Meaning | My value | Why |
|---|---|---|---|
| `rgba` | Subpixel stripe geometry (`rgb`/`bgr`/`vrgb`/`vbgr`/`none`) | `none` | Subpixel rendering assumes physical RGB stripe alignment; any bitmap rescale misaligns the color channels → fringing |
| `hintstyle` | Grid-fitting amount (`hintnone`/`hintslight`/`hintmedium`/`hintfull`) | `hintfull` | Crisp edges survive upscale better |
| `lcdfilter` | [FreeType LCD filter](http://freetype.org/freetype2/docs/reference/ft2-lcd_rendering.html) against ClearType fringing (`lcddefault`/`lcdlight`/`lcdlegacy`/`lcdnone`) | `none` | No subpixels → nothing to filter |

So `~/.config/fontconfig/fonts.conf` is grayscale + full hinting, mirrored in GNOME (`font-antialiasing grayscale`, `font-hinting full`). Under a fractional upscale this is the right combo: colored subpixel tricks can't survive resampling, but snapped grayscale edges can. Recommendation, not upstream doctrine — but the mechanics (per the [Arch wiki](https://wiki.archlinux.org/title/Font_configuration) and [FreeType LCD docs](http://freetype.org/freetype2/docs/reference/ft2-lcd_rendering.html)) point one way.

## Why a bigger font beats a smaller scale

[Ghostty](https://ghostty.org/docs/linux) on Linux is GTK/libadwaita with both Wayland and X11 backends, an [OpenGL renderer](https://ghostty.org/docs/), and — critically — it **re-renders on resize**, reallocating the GPU grid buffer (visible in logs as *"reallocating GPU buffer"*; resize stretch/flicker was a tracked bug class in [ghostty#7155](https://github.com/ghostty-org/ghostty/issues/7155) and [PR #7144](https://github.com/ghostty-org/ghostty/pull/7144)). No stretching: fullscreen gave me a freshly rasterized, larger grid — still passed through the same ~0.9 upscale. And [`font-size`](https://ghostty.org/docs/config/reference) is in points, scaled by display and text factors — at 13pt each glyph gets more source pixels, so the resample has more signal per stem. The blur is still mathematically there; it's just below perception now.

Verification commands I actually ran:

```bash
ghostty +show-config | grep -iE "font|theme|scale"   # live config, not the file you think you wrote
tr '\0' '\n' < /proc/$(pgrep -x ghostty)/environ | grep -E "GDK_BACKEND|WAYLAND_DISPLAY"
```

(`+show-config` is documented in the [Ghostty config docs](https://ghostty.org/docs/config); note the config file is `~/.config/ghostty/config`, not `config.ghostty` — the latter is silently ignored, a trap I fell into on day one.) One caution from my own notes: I once wrote `theme = Gruvbox Light` with a space and it errored — the name is `GruvboxLight`, no space.

## The fix ladder

In order, cheapest first:

1. **Confirm native Wayland** — `/proc/<pid>/environ` must show `WAYLAND_DISPLAY=wayland-0` and no `GDK_BACKEND=x11`. Kill stale launcher overrides.
2. **Grayscale + full hinting** — `fonts.conf` (`rgba none`, `hintfull`, `lcdfilter none`) plus matching `gsettings`. Costs nothing, removes fringing.
3. **Display size 100%** in ChromeOS Settings → Device → Displays — removes the upscale entirely. Costs screen real estate.
4. **Bigger `font-size`** (my fix: 12 → 13) — keeps your display-size preference, pushes glyphs above the blur threshold. `Ctrl+=` previews it live.
5. **`--scale`/`--dpi` on Sommelier** — last resort; you're hand-tuning the proxy.

Step 4 held for me on the built-in panel. Step 3 had already proven itself on the external monitor in the [flicker post]({% post_url 2026-10-20-ghostty-external-display-flicker %}). Same pipeline, same physics, two different cures — pick per display.

## Source

- [Sommelier README — proxy Wayland compositor](https://chromium.googlesource.com/chromiumos/platform2/+/main/vm_tools/sommelier/README.md) · [source dir](https://chromium.googlesource.com/chromiumos/platform2/+/main/vm_tools/sommelier) · [crosvm Wayland device](https://crosvm.dev/book/devices/wayland.html)
- [Exo README — host compositor, Crostini listed](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/components/exo/README.md) · [virtio-wayland transport](https://alyssa.is/using-virtio-wl/)
- [sommelier-x template — per-display Xwayland](https://chromium.googlesource.com/chromiumos/containers/cros-container-guest-tools/+/master/cros-sommelier/sommelier-x%40.service)
- [Linux on ChromeOS FAQ — fuzzy windows, Wayland preferred](https://chromeos.dev/en/linux/linux-on-chromeos-faq)
- [GNOME fractional scaling explainer](https://discourse.gnome.org/t/full-explanation-of-current-hidpi-fractional-and-integer-scaling-support-in-wayland/14225)
- [Arch font configuration — rgba/hintstyle/lcdfilter](https://wiki.archlinux.org/title/Font_configuration) · [FreeType LCD rendering](http://freetype.org/freetype2/docs/reference/ft2-lcd_rendering.html)
- [Ghostty keybind reference](https://ghostty.org/docs/config/keybind/reference) · [config reference](https://ghostty.org/docs/config/reference) · [config docs](https://ghostty.org/docs/config) · [Linux backends](https://ghostty.org/docs/linux)
- [VS Code blurry on ChromeOS#64070](https://github.com/Microsoft/vscode/issues/64070) · [Chromium 41462260](https://issues.chromium.org/41462260) · [chromium-os-discuss blur thread](https://groups.google.com/a/chromium.org/g/chromium-os-discuss/c/y7e0Ewuqleg)
- [Mesa llvmpipe](https://docs.mesa3d.org/drivers/llvmpipe.html)
