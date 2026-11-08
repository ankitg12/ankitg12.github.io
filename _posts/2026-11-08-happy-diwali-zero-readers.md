---
layout: post
title: "Happy Diwali to Zero Readers: Decorating Three Workspaces"
date: 2026-11-08
categories: personal productivity
---

My Desktop had 4 loose files, a 664-item Downloads folder sitting on it, and a wallpaper Finder answered with `missing value`. That is no way to welcome anyone — so I decorated three workspaces for Diwali: the desk, the Mac, and the agent context I work in.

Readers: zero, last I checked. My blog workflow says publish anyway. So — Shubh Deepavali. If you are reading this, you are reader number one.

## The three workspaces

| Workspace | Clean | Light | Pattern of welcome |
|---|---|---|---|
| Desk (physical) | wipe it, clear it fully | one diya (LED near cables) | marigold, one coin, one pen, one page |
| Mac | Desktop to zero loose files | warm wallpaper + Night Shift sunset-to-sunrise | `~/Pictures/Diwali-2026/`, Dock already at six tiles |
| Agent context | compress memory to one true sentence | one goal, not five | true project list, true goals, one journal line |

Order matters. Clean first, then light, then rangoli. In every workspace.

## Save state before you decorate

Same rule as debugging: capture before you reproduce. Before touching anything:

```bash
mkdir -p ~/Pictures/Diwali-2026 ~/Desktop/Pre-Diwali-Backup-2026-11-08
defaults read com.apple.dock persistent-apps > ~/Desktop/Pre-Diwali-Backup-2026-11-08/dock-apps.txt
ls -la ~/Desktop > ~/Desktop/Pre-Diwali-Backup-2026-11-08/desktop-ls.txt
osascript -e 'tell application "Finder" to get desktop picture'
```

Finder returned `missing value` for the wallpaper — worth knowing before I overwrote it, not after.

## Render the light, don't download it

I generated the wallpaper with stdlib-only Python first, then re-rendered with [Pillow](https://pillow.readthedocs.io/) once the install was allowed (`pip3 install Pillow` → `13.0.0`). Deep maroon to warm amber gradient, a diya glow at centre, rangoli arcs on a floor band, layered-ellipse flame:

```python
from PIL import Image, ImageDraw
W, H = 3024, 1964
img = Image.new("RGB", (W, H), "#2b0f14")
d = ImageDraw.Draw(img, "RGBA")
for y in range(H):
    t = y / (H - 1)
    if t < 0.6:
        k = t / 0.6
        r, g, b = int(43 + (122-43)*k), int(15 + (46-15)*k), int(20 + (28-20)*k)
    else:
        k = (t - 0.6) / 0.4
        r, g, b = int(122 + (240-122)*k), int(46 + (160-46)*k), int(28 + (70-28)*k)
    d.line([(0, y), (W, y)], fill=(r, g, b))
# ... glow rings, floor band, rangoli arcs, diya bowl, layered flame
img.save(os.path.expanduser("~/Pictures/Diwali-2026/diwali-diya.png"))
```

Set it:

```bash
osascript -e 'tell application "Finder" to set desktop picture to POSIX file (POSIX path of (path to pictures folder) & "Diwali-2026/diwali-diya.png")'
# exit=0
```

Warm light for the evening:

```bash
defaults write com.apple.nightshift schedule -string "sunset-to-sunrise"
defaults read com.apple.nightshift   # { schedule = "sunset-to-sunrise"; }
```

Then the sweep — 4 loose files off the Desktop into the backup folder. Raw output:

```
renamed '.../61UT6SSPo-L._SL1500_.jpg' -> '.../Pre-Diwali-Backup-2026-11-08/61UT6SSPo-L._SL1500_.jpg'
renamed '.../Screenshot 2026-10-22 at 11.41.11 AM.png' -> '.../Pre-Diwali-Backup-2026-11-08/...'
renamed '.../dock-backup-20261021.plist' -> '.../Pre-Diwali-Backup-2026-11-08/...'
renamed '.../file-:::.fileloc' -> '.../Pre-Diwali-Backup-2026-11-08/...'
```

Dock check: Apps, Edge, Notes, System Settings, Ghostty, Tern — six tiles, already minimal. Untouched.

## Why not the obvious alternatives

**Why not download a wallpaper?** A generated gradient is 42 KB, exact screen size, no licence question, no god photo. Which matters: never use god photos as wallpaper — icons sit on them. That is not respectful.

**Why not just write `defaults write` for Night Shift and call it done?** The preference writes fine, but on current macOS it may not drive the real Night Shift light. Verify in System Settings → Displays. A written pref is not a lit screen — same class of bug as [every layer said RUNNING while the speakers were silent](https://ankitg12.github.io/linux/debugging/audio/2026/11/03/speakers-silent-while-everything-said-running.html).

**Why not move the whole Downloads folder off the Desktop?** 664 items, possibly a working path. Sweeping blindly before a festival is how things break. Leave it; triage after Diwali.

## What didn't finish (honest list)

- Lock-screen text (`Shubh Deepavali — welcome Shri Lakshmiji`) needs sudo; `sudo -n` failed with `a password is required`. Needs an interactive terminal.
- The backup folder refuses to move off the Desktop: `mv: Resource deadlock avoided (os error 11)` to both Documents and Pictures. Something (Finder or iCloud Drive) holds it. A logout clears it.
- Downloads triage: 664 items, deferred deliberately.

Unfinished work is fine on Diwali evening. Note it, don't force it.

## Later threads

- Sort `~/Desktop/Downloads` after the festival; decide whether it is a working path or a pile.
- Move the backup folder off Desktop after a logout; confirm Desktop is truly bare.
- Confirm Night Shift actually engages at sunset in System Settings.
- Open the new-year build log tonight with one quiet task at the decorated desk.

Shubh Deepavali — to zero readers, and to reader number one.

## Source

- Wallpaper rendered with [Pillow](https://pillow.readthedocs.io/) (`pip3 install Pillow`), set via Finder `desktop picture` AppleScript
- Night Shift preference: `com.apple.nightshift schedule = sunset-to-sunrise` (verify in System Settings → Displays)
- Jekyll note: this post is date-stamped explicitly because [future-dated posts are silently dropped](https://jekyllrb.com/docs/posts/#writing-posts) — and Diwali waits for no build
