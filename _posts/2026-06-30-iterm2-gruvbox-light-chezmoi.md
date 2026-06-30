---
layout: post
title: "Making iTerm2 Gruvbox Light match WezTerm and Ghostty — a 6-minute war story with chezmoi, DynamicProfiles, and official presets"
date: 2026-06-30
categories: terminal productivity
series: "AI coding agent productivity"
---

**Problem**

I have Ghostty set to `theme = Gruvbox Light` and WezTerm using a custom `gruvbox_light` table ported from Windows Terminal (`#FBF1C7` cream background, `#282828` foreground, exact ANSI). iTerm2's built-in "Gruvbox Light" was close but not identical, and my initial custom `.itermcolors` + DynamicProfile only populated the dropdown — **Basic Colors stayed #FFFFFF/#000000**.

`chezmoi managed` kept saying "not managed", `cwd=~/tmp` turned relative `Library/...` paths into `~/tmp/Library/...`, and `Cmd+Q` + reopen didn't load the colors. The session ballooned to 80 tool calls.

**What worked**

1. **Official preset over custom plist**
   ```bash
   curl -sSL https://raw.githubusercontent.com/morhetz/gruvbox-contrib/master/iterm2/gruvbox-light.itermcolors \
     -o "$HOME/Library/Application Support/iTerm2/Color Presets/GruvboxLight.itermcolors"
   open "$HOME/Library/Application Support/iTerm2/Color Presets/GruvboxLight.itermcolors"
   ```
   `plutil -p` confirms `#FBF1C7` background. Filename-driven (no explicit "Color Preset Name" key).

2. **chezmoi with absolute target paths (the key fix)**
   ```bash
   chezmoi add "$HOME/Library/Application Support/iTerm2/Color Presets/GruvboxLight.itermcolors"
   chezmoi add "$HOME/Library/Application Support/iTerm2/DynamicProfiles/profiles.json"
   chezmoi apply "$HOME/Library/Application Support/iTerm2"
   ```
   `private_Library/...` in source maps correctly. Never `chezmoi add` paths inside `sourceDir`.

3. **Self-contained DynamicProfile**
   Embed full color dicts (not just `Custom Color Preset` name) so new windows don't need "Load Preset":
   ```json
   {
     "Profiles": [{
       "Guid": "12C59B15-BAFA-4898-87C4-309EFA5F06FA",
       "Name": "Gruvbox Light",
       "Normal Font": "JetBrainsMono Nerd Font 13",
       "Custom Color Preset": "GruvboxLight",
       "Use Bold Font": true,
       "Use Bright Bold": true,
       "Scrollback Lines": 100000,
       "Background Color": { "Red": 0.979, "Green": 0.936, "Blue": 0.733, "Color Space": "Calibrated" },
       ... (all Ansi + Foreground, Cursor, Selection)
     }]
   }
   ```

4. **Activation**
   ```bash
   defaults write com.googlecode.iterm2 "Default Bookmark Guid" -string "12C59B15-BAFA-4898-87C4-309EFA5F06FA"
   osascript -e '
     tell application "iTerm"
       activate
       create window with profile "Gruvbox Light"
     end tell
   '
   ```

**Lessons**
- Dropdown ≠ applied. **Load Preset** is mandatory for the current profile.
- `private_Library` + absolute `$HOME/Library` targets is the chezmoi pattern for macOS (relative paths + `~/tmp` cwd = silent failure).
- morhetz/gruvbox-contrib is the canonical source — our initial custom plist had sRGB vs Calibrated mismatch.
- `plutil -extract "Color Preset Name" raw ...` or `plutil -p` to debug naming.
- Managed skill [`iterm2-gruvbox-chezmoi`](skill://iterm2-gruvbox-chezmoi) now contains the full repeatable workflow.

**Result**
New iTerm2 windows open with exact parity to WezTerm (`Ctrl+Shift+T` toggle) and Ghostty (`theme = Gruvbox Light`). The 80-call war is over.

**Sources**
- [morhetz/gruvbox-contrib iterm2/gruvbox-light.itermcolors](https://github.com/morhetz/gruvbox-contrib/blob/master/iterm2/gruvbox-light.itermcolors)
- [iTerm2 Dynamic Profiles docs](https://iterm2.com/documentation-dynamic-profiles.html)
- [Chezmoi private_ discussion](https://github.com/twpayne/chezmoi/discussions/3843)
- Managed skill: `iterm2-gruvbox-chezmoi`

*Posted from the new iTerm2 window with Gruvbox Light active.*