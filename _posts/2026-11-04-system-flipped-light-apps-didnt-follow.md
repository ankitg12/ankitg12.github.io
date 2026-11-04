---
layout: post
title: "The System Flipped to Light. The Apps Didn't Follow."
date: 2026-11-04 09:30:00 +0530
categories: terminal productivity macos
series: "Agent Mishaps"
---

macOS Auto appearance flipped to light at sunrise. [Ghostty](https://ghostty.org/) stayed dark, [VS Code](https://code.visualstudio.com/) stayed dark, the [OMP](https://github.com/ankitg12/omp) marker still said `dark-volcanic`. The system toggle is not an app toggle — same class of miss as [tdark writing the theme while the window stayed light]({% post_url 2026-10-21-tdark-wrote-the-theme-the-window-stayed-light %}): the control you reached is not the control that draws the window.

## What macOS Auto does and doesn't do

Auto flips `AppleInterfaceStyleSwitchesAutomatically` and removes the `AppleInterfaceStyle` key (absent = light). Nothing else. No app re-reads that key on your behalf — each one needs its own write: Ghostty's `theme =` line, VS Code's `workbench.colorTheme`, OMP's both-slots marker + config, [Tern](https://docs.stencil.so/tern/)'s `theme_light`/`theme_dark` pins.

[Auto Dark Mode](https://autodarkmode.app/) solves exactly this layering on Windows — scheduled switch plus custom scripts into `tlight`/`tdark`. It is Windows-only by design ("macOS already has this"); on macOS the scheduler is built in, but the app-follow-through was missing.

## The follower: one path, all platforms

The shared `terminal-theme` dispatcher already handled Linux (gsettings, GNOME Terminal profiles). The mac additions needed no `is_macos` branch — every setter probes its own targets and no-ops elsewhere:

```python
def set_system_theme(mode: str):
    # gsettings present → GNOME, osascript present → macOS.
    # Each no-ops silently on the other platform.
    if "AUTO_FOLLOW" in os.environ:
        return
```

`AUTO_FOLLOW=1` is the key: the follower runs *after* macOS already flipped, so the osascript system toggle is skipped and only apps follow. The launchd agent polls every 5 minutes (lx01's `auto-theme.timer` polls every 15):

```bash
if defaults read -g AppleInterfaceStyle >/dev/null 2>&1; then
  WANT=dark
else
  WANT=light
fi
AUTO_FOLLOW=1 "$HOME/.local/bin/terminal-theme" "$WANT"
```

## The Ghostty name trap

The rename commit `bdfd7f6` switched to unspaced `GruvboxLight` — valid as a custom WezTerm scheme key, but Ghostty's bundled catalog (`Resources/ghostty/themes/`) ships `Gruvbox Light` with the space. Unspaced leaves Ghostty on fallback. Verified against the live catalog, restored spaced names in `a83623c`.

## Event-driven, not yet deployed

macOS posts `AppleInterfaceThemeChangedNotification` on `DistributedNotificationCenter` on every flip — [bouk/dark-mode-notify](https://github.com/bouk/dark-mode-notify) is the minimal Swift precedent. The `dark-mode-watch.swift` source in dotfiles subscribes to it and runs the same `AUTO_FOLLOW=1` path, which would kill the 5-minute poll lag. Compiles (`swiftc`, 58 KB binary), not yet wired to launchd — the poller is the running default until the watcher proves itself across sleep/wake.

## Why not a third-party scheduler

[NightOwl](https://nightowlapp.co/) and [Dynamic Dark Mode](https://github.com/ApolloZhu/Dynamic-Dark-Mode) flip the system on solar/manual schedules — but neither knows about OMP themes or Tern pins. The built-in Auto + a 20-line follower covers the same ground with zero new dependencies.

## Source

- [Auto Dark Mode](https://autodarkmode.app/) — Windows-only by design; custom scripts hook
- [bouk/dark-mode-notify](https://github.com/bouk/dark-mode-notify) — `AppleInterfaceThemeChangedNotification` listener pattern
- [Dynamic Dark Mode](https://github.com/ApolloZhu/Dynamic-Dark-Mode) — open-source macOS solar scheduler
- [NightOwl](https://nightowlapp.co/) — menu-bar scheduler with per-app exclusions
- [XTerm control sequences](https://invisible-island.net/xterm/ctlseqs/ctlseqs.html) — OSC 10/11/12 + OSC 4 palette
