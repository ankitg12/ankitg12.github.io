---
layout: post
title: "The Friction I Left Out: Four Weeks of GNOME Extensions on a Work Laptop"
date: 2026-10-14
categories: linux gnome productivity
series: "Moving to Linux"
---

This morning I opened my laptop and the brightness slider was gone. Twenty minutes later I had a Windows-style taskbar, a dead Super key, and a GNOME Shell that refused to load my start menu until I logged out.

The earlier posts in this series are mostly about wins: [three monitors that finally behave]({% post_url 2026-09-29-three-monitors-on-gnome-wayland %}), [a dictation indicator at the caret]({% post_url 2026-10-07-dictation-indicator-at-the-caret %}), [forty packages and one pipe]({% post_url 2026-10-13-the-linux-workbench-forty-packages-and-one-pipe %}). Those posts are true, but they leave out the friction. This post is the friction, taken from four weeks of my daily notes, so you can be ready for it.

## The setup

- Ubuntu 24.04 LTS, GNOME Shell 46, Wayland.
- A corporate laptop. I cannot move to a newer GNOME. The OS version is not my decision.
- Three external monitors (one in portrait, one 4K at 200%) plus the laptop panel.
- Extensions I use to make GNOME feel like the Windows desktop I came from:

| Extension | Why I use it |
|---|---|
| [Dash to Panel](https://extensions.gnome.org/extension/1160/dash-to-panel/) | One bottom taskbar instead of a top bar plus a dock |
| [ArcMenu](https://extensions.gnome.org/extension/3628/arcmenu/) | A start menu on the Super key |
| display-brightness-ddcutil | Brightness sliders for external monitors over DDC/CI |
| Tiling Shell | FancyZones-style window layouts |
| [smart-auto-move](https://github.com/khimaros/smart-auto-move) | Puts windows back on the same monitor after a restart |
| Hide Top Bar | Auto-hides the top bar |

Each one does what it promises on its own. The trouble shows up when they meet each other, a monitor change, a suspend, or a GNOME version their authors no longer test.

## Four weeks of friction, from my notes

| Date | What broke | Root cause | Cost |
|---|---|---|---|
| Sep 22 | Brightness extension saw no monitors | The packaged udev rule did not give my user access to the I²C bus of this APU's display controller | Custom udev rule |
| Sep 23 | Brightness slider stuck at "Initializing" | One monitor returned an invalid value for a power-state query (VCP `0xD6`) | A settings flag to skip the check |
| Sep 29 | Brightness slider dropped a random monitor | The extension probed all monitors in parallel and the probes collided | Local patch, **logout to load** |
| Sep 29 | Could not move the pointer up to the monitor above | Hide Top Bar's top-edge pressure barrier caught the pointer | Disabled the extension |
| Sep 29 | Gave up | — | Reverted to Ubuntu's default extensions |
| Oct 2 | Missed the start menu | — | Installed ArcMenu again, this time on Ubuntu Dock |
| Oct 6 | Brightness went to 0 while I dragged the slider | Overlapping write processes fought over one I²C bus lock | Serialized writes; sent upstream as [PR #196](https://github.com/daitj/gnome-display-brightness-ddcutil/pull/196) |
| Oct 6 | One monitor missing from the slider | A detection cache written before that monitor was connected | Deleted the cache |
| Oct 7 | My own caret-indicator extension | New extension code loads only in a new Shell | **Two logouts** in one hour |
| Oct 8 | **The whole session ended** while a script opened console windows | smart-auto-move put a window back on "monitor 3", which did not exist at that moment. Mutter hit an assertion and GNOME Shell aborted. | Lost every open window. Upstream had fixed it in a version for GNOME 49 and later only, so I backported a one-line bounds check. **Logout to load.** |
| Oct 14 | After resume: top bar hidden, brightness icon gone | Three reloads in quick succession after the monitors woke | Tried Dash to Panel again |
| Oct 14 | Super key dead after enabling Dash to Panel | ArcMenu crashed: `TypeError: this.panel is undefined` | **Logout** |

Most of these were not hard to diagnose once I read the log. The cost is that each one stopped my real work, and the fix often ended in a logout.

## Why the logout keeps coming back

The single fact that explains most of this table: **on Wayland, GNOME Shell is the display server.**

On X11, the Shell was only a window manager. You could press Alt+F2, type `r`, and the Shell restarted while your apps kept running. The [GNOME extension guide](https://gjs.guide/extensions/development/creating.html) still describes this restart for X11 sessions. For Wayland it suggests a nested Shell in a window for testing, because there is nothing to restart in place. The Shell, the compositor, and your session are one process.

Three things follow from that:

1. **An extension crash can end your session.** Extensions are JavaScript that runs inside the Shell process and patches its internals. On Oct 8, one missing bounds check in an extension turned into `SIGABRT` in the compositor, and everything I had open was gone.
2. **A crashed extension stays crashed.** When ArcMenu threw its error this morning, GNOME marked it `ERROR`. Turning it off and on did not help. Installing it again through the Shell's own D-Bus installer gave a new error, `ArcMenu has been already initialized`, because ArcMenu keeps a single-instance flag in the Shell's memory. Only a new Shell clears that flag, and on Wayland a new Shell means logging out.
3. **New code loads only at login.** Upgrading or patching an extension, or unpacking one with `gnome-extensions install`, does not reach the running Shell. When I unpacked Dash to Panel and enabled it, the answer was:

   ```
   $ gnome-extensions enable dash-to-panel@jderose9.github.com
   Extension "dash-to-panel@jderose9.github.com" does not exist
   ```

   The files were on disk. The running Shell had never scanned them.

## This morning's crash, step by step

The trace is worth reading because it shows how extensions break each other:

```
TypeError: this.panel is undefined
  _configureActivitiesButton@.../arcmenu@arcmenu.com/menuController.js:417
  MenuController@.../menuController.js:54
  _enableButtons@.../arcmenu@arcmenu.com/extension.js:206
```

1. ArcMenu watches for Dash to Panel. When Dash to Panel is enabled, ArcMenu rebuilds its button on Dash to Panel's taskbars.
2. Dash to Panel does not build its taskbars at once. When Ubuntu Dock is running, it first disables Ubuntu Dock and then **waits 200 ms** before it finishes enabling.
3. ArcMenu rebuilt during that gap. It found no Dash to Panel taskbars, fell back to the stock top bar, and looked for a property that only Dash to Panel's taskbar objects have. Crash.

Neither extension is wrong on its own. Each one is reasonable code that assumes something about the other's timing. This is a race, and it happens only when you enable Dash to Panel in a running session while Ubuntu Dock is active. At login, Ubuntu Dock is already off, so the 200 ms wait does not happen. After the logout, the Super key worked again.

One more surprise: after the logout, `gnome-extensions info arcmenu@arcmenu.com` still said `State: ERROR`, even though the menu worked. A second enable call during login hit the "already initialized" guard. **The state that GNOME reports and what you see on screen can disagree.**

## The LTS gap

Some of this comes from running an older GNOME. Ubuntu 24.04 is a long-term release and stays on GNOME 46. Extension authors work against the newest GNOME. When smart-auto-move fixed the crash that ended my session, the fix went into the builds for GNOME 49 and 50. The GNOME 46 build did not get it.

On a personal machine I would just upgrade. On a corporate laptop the OS image is not mine to change, and I suspect that is true for most people who move to Linux at work. Plan for it: **the fix you need may already exist upstream, for a GNOME version you cannot have.** Sometimes you can backport it yourself (a one-line bounds check was enough here). Sometimes you live without it.

## Be ready: what I do now

1. **Save your work before you enable an extension.** Assume the next step is a logout.
2. **Enable extensions at login, not in a running session.** Change the enabled list, then log out once. That avoids timing races like the ArcMenu one.
3. **Keep a known-good list and a way back:**

   ```bash
   gsettings get org.gnome.shell enabled-extensions > ~/extensions-known-good.txt
   # to restore later:
   gsettings set org.gnome.shell enabled-extensions "$(cat ~/extensions-known-good.txt)"
   ```

4. **Do not trust `State:` alone. Ask the Shell for the error:**

   ```bash
   gdbus call --session --dest org.gnome.Shell.Extensions \
     --object-path /org/gnome/Shell/Extensions \
     --method org.gnome.Shell.Extensions.GetExtensionErrors arcmenu@arcmenu.com
   journalctl --user -b -o cat | grep -A8 "JS ERROR"
   ```

5. **To load a new extension without logging out,** use the Shell's own installer instead of unpacking files yourself. It shows a confirmation dialog and loads the extension into the running Shell:

   ```bash
   gdbus call --session --dest org.gnome.Shell.Extensions \
     --object-path /org/gnome/Shell/Extensions \
     --method org.gnome.Shell.Extensions.InstallRemoteExtension dash-to-panel@jderose9.github.com
   ```

6. **Watch extensions that move windows between monitors.** If you dock and undock, their saved positions can point at a monitor that is not there. In my notes, that is the only class of bug that took down the whole session.
7. **Count your extensions.** Each one is code inside your compositor. Before you add one, ask whether stock GNOME already does it, even if not quite the way you want.

## The lineage

Firefox went through this exact problem. Its legacy XUL add-ons could reach into the browser's internals, which made them powerful and fragile: they broke when the browser changed and could break each other. In [Firefox 57 (2017)](https://blog.mozilla.org/addons/2017/09/28/webextensions-in-firefox-57/), "legacy add-ons no longer load". Mozilla replaced them with WebExtensions, which can only use a fixed, versioned API. Users lost some powerful add-ons, and in return their browser stopped breaking on every update.

GNOME extensions today sit where XUL add-ons sat before 2017: they monkey-patch a live process, with no stable contract between the Shell and the extensions, or between one extension and another. Until GNOME draws that line, every extension you add is a small bet that its author tested your exact Shell version, your exact neighbours, and your exact monitors.

I am still on Linux, and I would make the move again. But "it just works" was never true for me. The desktop I wanted came from six extensions, and I paid for it in broken sliders, a lost session, a dead Super key, and more logouts than I can count. Know that before you start, and budget for it.

## Source

- [Dash to Panel](https://extensions.gnome.org/extension/1160/dash-to-panel/) and [ArcMenu](https://extensions.gnome.org/extension/3628/arcmenu/) on extensions.gnome.org
- [display-brightness-ddcutil PR #196](https://github.com/daitj/gnome-display-brightness-ddcutil/pull/196): skip invalid displays, serialize probes and writes
- [smart-auto-move](https://github.com/khimaros/smart-auto-move)
- [GNOME Shell extension guide: creating and testing](https://gjs.guide/extensions/development/creating.html)
- [WebExtensions in Firefox 57](https://blog.mozilla.org/addons/2017/09/28/webextensions-in-firefox-57/), Mozilla Add-ons blog
