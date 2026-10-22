---
layout: post
title: "My Mac froze after logout. The logs said it wasn't the logout."
date: 2026-10-22
categories: macos debugging
---

My Mac froze at 15:49. I had logged out of a second account and was heading back to mine. Screen dead, cursor dead, power button the only way out — or so I thought. Twenty minutes of log-diving later, the story the machine told was not the one I walked in with, and the difference is the whole post.

## What I thought happened

Fast User Switching hung. I was on account B, switching back to account A, and the switch wedged. Clean theory. Wrong in nearly every word.

## What the log says

Timeline from `loginwindow` (pid 13401), all IST:

```
15:49:32  kAELogOutEvent, session 100189, uid 502
15:49:34  confirmed (result 1000) — ~2s, not the 60s timer
15:49:35  LogoutComplete
15:49:36  login window up, Security Agent started
15:49:48  "transition to the desktop", _fusInProgress:0
15:49:48  WindowManager: Invalid connection
15:49:50  old session still dying (Karabiner death, HID flap)
15:50:05  Stretchly long break fires into the transition
15:51:13  button reset (telemetry: btn_rst, btn_seq_reset, no panic)
```

Logout finished. The login window appeared. The freeze is the handoff from login window back to desktop — and the system's own flag, `_fusInProgress:0`, says Fast User Switching was not involved. I kept saying "switch back" for most of the session. The machine disagreed.

## The session zoo, since I had to learn it

macOS has four different "somebody else's screen" states and they fail differently:

- **Lock Screen** — same account, screen locked. Cheapest. Session never dies.
- **Fast User Switching** — second account on screen, first still logged in behind it. Both sessions alive. This is the Apple-menu-username / Control Center path.
- **Logout** — session ends, apps quit, login window. What I actually did.
- **Login window → desktop transition** — the handoff. This is where mine died: `WMClientWindowManager: Invalid connection` and 85 seconds of nothing.

Knowing which of the four you are in decides which logs matter. I spent the first stretch grepping for switch events that never existed.

## Three suspects, three verdicts

**Fast User Switching: acquitted.** Flag says 0. Moving on.

**`set_cursor_surface` errors at 15:49:00: symptom, not cause.** WindowServer logged `Cursor disabled: failed set_cursor_surface` three times before I even clicked logout — and the same line recurs after the reboot (new WindowServer pid, healthy box). It fires on a working machine. It told me the display stack was already unhappy, not what killed it.

**Stretchly :50 long break: bad timing, not the killer.** My clock-aligned break scheduler fired a fullscreen long break 17 seconds into the hung transition. My `gui/501` LaunchAgent runs even when I am not the console user, so the overlay landed on top of a wedged WindowServer. Guilty of piling on, innocent of starting it. Fix: five-line console-owner guard — skip unless `stat -f%Su /dev/console` is me:

```bash
ts() { date "+%Y-%m-%d %H:%M:%S"; }   # local timestamp helper in the script
OWNER="$(stat -f%Su /dev/console 2>/dev/null || echo unknown)"
if [ "$OWNER" != "ankitg" ]; then
  echo "$(ts)  SKIP  console-owner=$OWNER" >>"$LOG"
  exit 0
fi
```

## What I did not prove

Force restart recovered the box. I briefly claimed it was *the* recovery. It is not — it is the recovery I used. Untested: the `Cmd+Opt+Esc` force-quit chord (if WindowServer still routed keys), a second machine over Screen Sharing/SSH (SSH was off on this box, so that path did not exist today), and simply waiting past 85 seconds. One hang is a note, not a conclusion.

## Adjacent reports

Same neighborhood, none a confirmed match for 27.0.1. Apple's [macOS 27 release notes](https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes) and [27.2 RC notes](https://developer.apple.com/documentation/macos-release-notes/macos-27_2-release-notes) list no login-transition issue. Tahoe-era login-path freezes: [Apple Communities screensaver/lock thread](https://discussions.apple.com/thread/256165873), [fast user switching slowdown thread](https://discussions.apple.com/thread/256155127), [r/MacOS Tahoe account-switch thread](https://www.reddit.com/r/MacOS/comments/1ntdcda/another_tahoe_bug_cant_switch_between_accounts/), [r/macsysadmin black screen after login](https://www.reddit.com/r/macsysadmin/comments/1r7np31/macos_black_screen_after_logging_in/), [r/macsysadmin fleet frozen at login](https://www.reddit.com/r/macsysadmin/comments/1tedkil/issue_fleet_frozen_after_any_login_attempt/), [cursor freeze after fast user switch](https://apple.stackexchange.com/questions/486588/macos-cursor-ui-freezes-after-fast-user-switching), [Hackintosh blank-after-login](https://www.insanelymac.com/forum/topic/362155-screen-goes-blank-intermittently-after-login/).

## Next time, in order

1. Key chords first — `Cmd+Opt+Esc`, then `Ctrl+Opt+Cmd+Esc` for Finder.
2. Turn on Remote Login for admins *now*, test it from another machine *before* you need it. A wedged login window still answers on port 22.
3. If it hangs identically a second time: `sudo sysdiagnose -f ~/freeze2` over SSH, then reset. Two matching hangs with sysdiagnose is a fileable report.

## The method that transfers

Timeline first (`log show --start --end --predicate 'process == "loginwindow"'`), per-suspect grep second, and let the machine's own flags (`_fusInProgress`, telemetry reset causes) overrule your memory of what you clicked. The 20 minutes were not wasted: three theories died, one guard shipped, and the playbook above is now written down instead of rattling around in my head.
