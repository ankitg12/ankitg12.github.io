---
layout: post
title: "Ghostty Flickered on the External Display. The Fix Was 100% Scale, Not the Backend."
date: 2026-10-20
categories: linux wayland debugging
series: "Moving to Linux"
---

Dragging my Ghostty window from the Chromebook's internal display to the external monitor turned it into a strobe light.

## The setup

ChromeOS Crostini container (Debian 12), Ghostty 1.1.3 running native Wayland through the Sommelier bridge. Two outputs, two different logical sizes, side by side:

```text
$ xrandr
Screen 0: minimum 16 x 16, current 3262 x 981
XWAYLAND0 connected 1512x853+1745+0    # internal
XWAYLAND1 connected 1744x981+0+0       # external
```

No `/dev/dri` in the container — the GL stack (Mesa/virgl, GTK4/GSK OpenGL renderer) has no device node behind it. That matters later.

## The wrong fix I tried first

My other machine (lx01) had a Ghostty flicker fixed with `GDK_BACKEND=x11` — force Xwayland, let the compositor handle the surface. So I patched the Chromebook launcher:

```ini
Exec=env GDK_BACKEND=x11 ghostty
```

Same fix, same symptom, wrong call. On lx01 the flicker was a GNOME/Mutter multi-scale negotiation problem where Xwayland was strictly steadier. Here, Xwayland through Sommelier at a fractional ChromeOS scale means the compositor resamples every frame — stable, but soft. I wanted sharp, and I wanted Ghostty to live on the external display only. Xwayland buys stability by giving up the exact thing I asked for:

| Backend | Sharp at fractional scale | Stable across mixed scales |
|---|---|---|
| Native Wayland | Yes — pixel-exact at 100% | Flickers on the scale crossing |
| Xwayland (`GDK_BACKEND=x11`) | No — compositor resamples | Stable |

So I reverted the launcher. The backend was never the problem; the scale crossing was.

## The actual fix

Three steps, all host-side or placement, zero config:

1. **ChromeOS Settings → Device → Displays → external display → Display size to minimum (100%).** Integer scale means Sommelier reports a clean 1:1 viewport — the Wayland surface maps pixel-exact, no fractional renegotiation, no resampling.
2. **Move the shelf to the external display** so Ghostty spawns there directly.
3. **Park Ghostty on the external and leave it there.** It never crosses the mixed-scale boundary, so the flicker path never runs. Native Wayland stays sharp.

Flicker gone immediately. No renderer flags, no launcher overrides, `Exec=ghostty` untouched.

## Why this works

Native Wayland renegotiates buffers and fractional scale every time the window crosses between differently-scaled outputs — through a GL stack with no `/dev/dri` behind it, in a bridge (Sommelier `--stable-scaling`) that exists precisely because ChromeOS scales displays fractionally. At 100% on the parked display there is nothing to renegotiate: one scale, one surface, no crossing. The flicker was the crossing, not the renderer.

## Gotchas

- **Ghostty is single-instance.** The old backend keeps serving new windows until every window is closed. Close them all before any relaunch-and-test, or you're testing the old process.
- **Launcher overrides only apply at spawn, from the launcher.** `Exec=env GDK_BACKEND=x11 ghostty` does nothing if you start Ghostty from a terminal. Shelf launch is the reliable path — which is also why step 2 matters.
- **The internal display at 100% looks small.** Leave it wherever it is; Ghostty doesn't live there anymore.
- **Dragging the window back to the internal display may flicker again.** That is the same mixed-scale crossing, not a regression.
- **If it still flickers parked at 100%,** the next suspect is the GL renderer itself (no `/dev/dri`), and the next step is `GSK_RENDERER=cairo` (software rendering) in the launcher. Sharp stays; GPU acceleration goes.

## Source

- [Ghostty on Linux](https://ghostty.org/docs/linux) — distro packages and platform notes
- [Prebuilt binaries and packages](https://ghostty.org/docs/install/binary) — why bookworm has no official package (I run a community `.deb` from [dariogriffo/ghostty-debian](https://github.com/dariogriffo/ghostty-debian))
- Related: {% post_url 2026-10-16-xwayland-authorization-required-stale-xauthority %} — the lx01 Xwayland override this post deliberately did *not* reuse
