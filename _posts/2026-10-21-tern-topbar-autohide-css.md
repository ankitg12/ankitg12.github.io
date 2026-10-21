---
layout: post
title: "Hiding Tern's Top Bar: 55px Back with Three Lines of CSS"
date: 2026-10-21
categories: terminal css productivity
---

Tern's vertical tabs already auto-hide. Its top tab row doesn't — a 58px strip duplicating the sidebar rail, with no setting to collapse it. `tabs_autohide` exists; `topbar_autohide` doesn't. Fullscreen is all-or-nothing. So I patched it with CSS through Tern's own control socket.

## The target

`tern ctl dump 'nav#tabbar'` says it plainly:

```json
{"path": "nav#tabbar", "rect": [11.69, 0.0, 1428.31, 58.47], "visible": true}
```

58px tall at the top of the window, even in vertical-tabs mode. Tern exposes live CSS injection:

```bash
tern ctl css topbar-autohide ~/helpers/tern-topbar-autohide.css
```

Sheets apply instantly, no restart. Removing the path arg (`tern ctl css topbar-autohide`) drops the sheet.

## Three versions that failed

| v | Approach | Result |
|---|----------|--------|
| 1 | `max-height: 8px; opacity: 0.15` | Collapsed to 12px but the box still reserved space. |
| 2 | `display: none` + `body::before:hover` reveal strip | Space freed (rect 0×0) but the parser rejected the pseudo-element selector — no way back except dropping the patch. |
| 3 | `display: none` + `body:hover nav#tabbar` | Parses clean, but `body:hover` is true whenever the pointer is anywhere in the window. Bar always visible. Useless. |

The lesson: `display:none` kills the hover target, and there's no CSS parent-hover that limits to the top edge. Dead end for pure show/hide.

## What worked: get out of flow

Two moves. First, pull the bar out of document flow so it stops reserving its row — absolute position, parked above the viewport, overlaying on hover instead of pushing content down. Second, the real find: the 55px gap wasn't the bar at all. `main#panel` starts at y=55 — a fixed offset reserved above the panel. Shifting the panel up with a compensating padding reclaims it:

```css
/* ~/helpers/tern-topbar-autohide.css */
main#panel {
  margin-top: -55px !important;
  padding-top: 55px !important;
}
nav#tabbar {
  position: absolute !important;
  top: -58px !important;
  left: 0 !important;
  right: 0 !important;
  opacity: 0 !important;
  z-index: 9999 !important;
  transition: top 0.18s ease, opacity 0.18s ease !important;
}
nav#tabbar:hover {
  top: 0 !important;
  opacity: 1 !important;
}
```

Measured via `tern ctl dump`:

```json
// main#panel before → after
[11.69, 55.47, 1425.31, 841.53] → [11.69, 0.47, 1425.31, 896.53]
```

+55px of terminal height. Hover the top edge and the tab row drops over the content.

The negative-margin idea came from [firefox-autohide-navbar](https://github.com/3ae3ae/firefox-autohide-navbar) — same pattern of keeping the element hoverable while hiding it, adapted from margin-overlap to absolute-overlay because Tern's layout reserves the row above the panel rather than inside it.

## Caveats

- **Update (verified): bundle it.** `styles = ["topbar.css"]` in the window plugin's `plugin.toml` installs the sheet at every `window_start` — survives full quit/reopen, no re-apply command. Restart-tested.
- The macOS traffic-lights/titlebar strip above it is OS chrome; no CSS reaches it. Fullscreen (`Cmd+Ctrl+F`) is the only move there.
- This is a local workaround, filed upstream as [stencil-hq/tern-sdk#129](https://github.com/stencil-hq/tern-sdk/issues/129) with the full CSS. A native `topbar_autohide` needs position/JS-driven reveal — pure show/hide can't do edge-hover, as v2/v3 proved.

## Source

- [stencil-hq/tern-sdk#129](https://github.com/stencil-hq/tern-sdk/issues/129) — feature request + workaround
- [3ae3ae/firefox-autohide-navbar](https://github.com/3ae3ae/firefox-autohide-navbar) — the negative-margin autohide pattern
