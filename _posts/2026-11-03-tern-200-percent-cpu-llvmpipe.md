---
layout: post
title: "My terminal was burning 200% CPU. The fix was 16 lines of JSON."
date: 2026-11-03 10:00:00 +0530
series: "AI coding agent productivity"
---

My machine was unusable. Typing lagged, everything stuttered. `top` said the terminal emulator itself — [Tern](https://github.com/tern-tools/tern) — was eating 200%+ CPU on a 2-vCPU Chromebook container. Not a build, not a test suite. The window I was typing in.

The fix was 16 settings. Here's how I found them.

## The numbers

| | Before | After |
|---|---|---|
| `tern` CPU | 200%+ (both cores) | ~150% under active use |
| Frame encode p50 | 944 ms | 101 ms |
| Load average | 2.0 | 1.7 and falling |
| Typing | unusable | works |

Nine-fold drop in per-frame cost. The load is still above 1 because typing *produces* frames — but each frame now costs a tenth of what it did.

## Why: software rendering

The container has no GPU device (`/dev/dri` doesn't exist), so every frame goes through [llvmpipe](https://docs.mesa3d.org/drivers/llvmpipe.html) — [Mesa's](https://www.mesa3d.org/) software rasterizer. Tern's own log said it plainly:

```
software rendering on llvmpipe: no usable GPU driver,
so every frame is drawn on the CPU
```

On real hardware, blur radii, translucency, and anti-aliasing are nearly free — the GPU absorbs them. On llvmpipe, each one is CPU cycles per pixel per frame. [Ghostty](https://github.com/ghostty-org/ghostty) documents the same tradeoff class in its renderer (see e.g. [its Metal backend](https://github.com/ghostty-org/ghostty/blob/main/src/renderer/Metal.zig)): GPU terminals assume a GPU. Take it away and every decorative effect becomes a tax.

## Finding the cost centers

Tern ships a `ctl perf` command that breaks down frame time. The breakdown showed **encode at 99% of frame cost** — rasterization, not layout, not the agent, not I/O. `vmstat` confirmed: 72–93% user CPU, zero iowait, zero swap. Pure pixel-pushing.

The animating-elements list showed the repaint drivers: a working-tab spinner, a blinking caret, and three chat-reader regions re-rasterizing a 1400-node transcript on every frame.

## The 16 lines

```json
{
  "material": "Solid",
  "blur": 0,
  "opacity": 100,
  "sync_to_display": true,
  "frames_queued": 1,
  "auto_update": false,
  "native_prompt": false,
  "native_completion": false,
  "scrollback_lines": 2000,
  "confirm_close": "Never",
  "surface_chat": "Console",
  "antialias": false,
  "snap_text": true,
  "density": "Compact",
  "expand_thoughts": false
}
```

What each group does:

- **Compositor effects** (`Solid`, `blur: 0`, `opacity: 100`) — kills the backdrop blur and translucency. Biggest single win: blur is a convolution over every background pixel, on the CPU.
- **Frame pipeline** (`sync_to_display`, `frames_queued: 1`) — vsync on, one frame queued. Stops the renderer from preparing frames nobody will see.
- **Background work** (`auto_update: false`) — no silent update checks competing for the same two cores.
- **Native surfaces off** (`native_prompt`, `native_completion: false`) — the chat transcript renders as plain console text instead of re-rasterized rich surfaces. This is what killed the 99%-encode frames.
- **Less to draw** (`scrollback_lines: 2000`, `Compact`, `expand_thoughts: false`) — smaller transcript, denser layout, folded reasoning blocks.
- **No AA** (`antialias: false`, `snap_text: true`) — glyphs snap to whole pixels; no analytic edge smoothing per frame.
- **`confirm_close: Never`** — unrelated to speed, but the close-confirmation dialog hung without a dialog tool installed, wedging the event loop. Worth setting on minimal containers.

Applied by restarting the Tern window (settings load at startup; the daemon holds the session, so no work is lost).

## The constraint that mattered

One requirement shaped everything: the window stays full-screen. No "shrink it" cop-out — on a 1280×800 display that's ~1M pixels per frame through llvmpipe regardless. All savings had to come from *what* gets drawn, not *how much screen* it covers.

## Takeaway

When a GPU terminal runs on a CPU, audit every pixel effect. `ctl perf`-style frame breakdowns tell you in one reading whether you're layout-bound or raster-bound — and if it's raster, translucency and blur go first. The decorative settings are the expensive ones precisely because they're invisible in their cost on real hardware.

## Source

- [Tern](https://github.com/tern-tools/tern) — `ctl perf`, `ctl stats` for frame diagnosis
- [llvmpipe driver docs](https://docs.mesa3d.org/drivers/llvmpipe.html) — why every frame is CPU-drawn
- [Mesa](https://www.mesa3d.org/) — software rendering stack
- [Ghostty](https://github.com/ghostty-org/ghostty) — GPU-terminal renderer tradeoffs; [Metal backend](https://github.com/ghostty-org/ghostty/blob/main/src/renderer/Metal.zig)
