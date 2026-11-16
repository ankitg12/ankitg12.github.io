---
layout: post
title: "Print Screen Dimmed My Desktop. Escape Did Nothing."
date: 2026-11-16
categories: linux gnome debugging
series: "Moving to Linux"
---

I pressed Print Screen, and my desktop stayed dim with a bright rectangle over one monitor. Escape did nothing.

The desktop was [GNOME](https://www.gnome.org/) on [Ubuntu](https://ubuntu.com/desktop), with three monitors. A full-desktop screenshot showed a lot of black space between the displays, so my agent cropped the image to the affected monitor. That made the overlay clear: the terminal was dim, with a lighter rectangle over it.

The agent initially called it a screenshot-selection overlay. That was plausible, but not established. Sending Escape through a virtual keyboard and pressing Print Screen again did not remove it.

## The recording that was still running

A process check found GNOME's screen recorder. A file in `~/Videos/Screencasts` was still open and growing:

```text
COMMAND  USER    FD  TYPE  SIZE/OFF  NAME
gjs      <user>  22w  REG  17216697  <recording>.webm
```

The number is the file size observed at one point, not the duration or a diagnosis. The open write descriptor and later file updates showed that the recording was still active.

The agent tried the recorder's [D-Bus](https://www.freedesktop.org/wiki/Software/dbus/) stop method. It returned `false`. The installed recorder source explains that result: `StopScreencast` looks up a recorder by the calling D-Bus sender, and returns `false` if that sender has no recorder. A separate command was not the client that started this recording.

## The shortcut did not resolve it

Other people have reported [GNOME recordings that do not stop](https://discussion.fedoraproject.org/t/cant-stop-gnome-shell-screencast-on-fedora38/84397/2). Ubuntu's [screen recording guide](https://ubuntu.com/desktop/docs/en/latest/how-to/record-the-screen/) says to stop a recording using its red panel indicator; the discussion also mentions `Ctrl+Alt+Shift+R`.

The agent sent that shortcut. A screenshot in a dark terminal theme appeared to show the overlay had disappeared, but changing back to a light theme made it visible again. The same recording file remained open and continued to grow. Neither the overlay nor the recording was resolved.

## What this did not establish

- Whether the recording caused the overlay, or whether both came from the same interaction.
- Why the overlay was difficult to see in the dark theme, or how to dismiss it.
- How to stop this recording cleanly from a different D-Bus client.

A dark theme made the screen look normal in the screenshot. Changing the theme showed that the overlay was still there. The recording was still active too.

## Source

- [Ubuntu: Record the screen](https://ubuntu.com/desktop/docs/en/latest/how-to/record-the-screen/)
- [Fedora Discussion: Can't stop GNOME Shell screencast](https://discussion.fedoraproject.org/t/cant-stop-gnome-shell-screencast-on-fedora38/84397/2)
- [GNOME Shell source](https://gitlab.gnome.org/GNOME/gnome-shell)
