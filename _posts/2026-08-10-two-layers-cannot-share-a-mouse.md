---
layout: post
title: "Two Layers Cannot Share a Mouse"
date: 2026-08-10
categories: terminal tooling agents
---

My terminal emulator has a custom right-click menu — open the selected text in an editor, search it, hand it to an agent. It stopped working the moment I started running a terminal multiplexer for coding agents inside it.

The obvious diagnosis is a broken key binding. It is not. It is a question of which of two programs owns each mouse gesture, and the answer turns out to be forced by where the scrollback buffer lives.

---

## Why the binding silently stops firing

A terminal emulator draws the window and owns the mouse. A full-screen terminal application (a multiplexer, `vim`, `htop`) can ask to *borrow* it, by enabling mouse reporting:

```
CSI ? 1002 h    # report button press/release and drag
CSI ? 1006 h    # SGR encoding — coordinates beyond column 223
```

Once reporting is on, the emulator forwards mouse events to the application instead of acting on them. Every ordinary binding in your config becomes dead code inside that pane. WezTerm makes the escape hatch explicit — a binding must opt in:

```lua
{
  event           = { Up = { streak = 1, button = 'Right' } },
  mods            = '',
  mouse_reporting = true,          -- also fire while the app has the mouse
  action          = wezterm.action_callback(on_right_click),
},
```

## The trap: bind `Up`, but neutralise `Down`

With that in place my menu appeared — but only on empty space. Right-clicking *selected text* just copied it and showed nothing.

The cause is ordering. WezTerm clears the selection on button-**down**. If only `Up` is bound, the callback runs after the selection is already gone, reads an empty string, and falls through to whatever the no-selection branch does. The menu never had a chance.

Every `Up` binding that inspects the selection needs a matching `Down` that does nothing:

```lua
{ event = { Down = { streak = 1, button = 'Right' } }, mods = '', action = act.Nop },
{ event = { Up   = { streak = 1, button = 'Right' } }, mods = '', action = cb   },
```

Bind the action on `Up` so the overlay is not dismissed by the release; bind `Down` to `Nop` so the selection survives long enough to be read. Both, or neither works.

Verify without dumping the entire keymap into your terminal:

```bash
wezterm show-keys | sed -n '/Mouse: mouse_reporting/,$p'
```

```
CTRL    Down { streak: 1, button: Left }    ->   Nop
SHIFT   Down { streak: 1, button: Right }   ->   Nop
CTRL    Up { streak: 1, button: Left }      ->   OpenLinkAtMouseCursor
SHIFT   Up { streak: 1, button: Right }     ->   EmitEvent("user-defined-1")
```

## The part that is not configurable

Menus solved, I wanted the other half: plain left-click on a file path should open it, exactly as in a bare terminal tab. The multiplexer offers a switch for precisely this — stop capturing the mouse, let the terminal handle ordinary clicks.

It works. It also breaks scrolling.

| mouse capture | click opens links | wheel scrolls the pane | pane focus, split drag, app menu |
|---|---|---|---|
| on (default) | no — needs a modifier | yes | yes |
| off | yes | **no** | no |

This is not a missing feature. **The pane's scrollback lives inside the multiplexer**, in its own ring buffer; the outer terminal has never seen those lines and cannot scroll them. So whichever layer receives the wheel must also receive the click — they arrive through the same reporting stream. Bare-click-to-open and multiplexer scrollback are mutually exclusive by construction.

Once that is clear, the design question answers itself: give the inner app the bare gestures it needs to function, and put the outer terminal's extras on modifiers.

| gesture | owner |
|---|---|
| left-click, wheel, drag | multiplexer — focus, scrollback, resize |
| `Ctrl`+left-click | terminal — open the resource under the pointer |
| right-click | multiplexer — its own pane menu |
| `Shift`+right-click | terminal — text-action menu |

Modifier choice is not free: terminals commonly reserve `Shift`+mouse to bypass reporting, which is exactly why it is the natural home for "I meant the terminal, not the app."

## The finding that was worth the afternoon

`Ctrl`+click now opens links inside the multiplexer using the emulator's `hyperlink_rules` — including bare Windows paths and `~/` paths, which the multiplexer itself will never make clickable. Its detector is prefix-matched, not regex:

```rust
// src/app/actions.rs
pub(crate) fn safe_web_url(url: &str) -> Option<&str> {
    (url.starts_with("http://") || url.starts_with("https://")).then_some(url)
}
```

Literal `http://` and `https://`, plus OSC 8 escape hyperlinks. No pattern configuration exists anywhere in its config model.

The interesting part: the capability is already vendored. The project embeds Ghostty's VT engine, whose configuration supports user-defined `link` regexes with priority ordering. The feature is present one layer down and simply not surfaced — which makes "expose it" a far smaller and more honest patch than "write a link matcher."

## Two lessons, one uncomfortable

**Read the source before arbitrating the symptom.** I spent an hour negotiating gestures between two layers. A single `grep` for the URL matcher, and two minutes in the upstream issue tracker, would have reframed the whole exercise at the start — including discovering that maintainers had a live branch touching the same hit-testing code, and a string of closed bugs about `Ctrl`+click in split and mouse-reporting panes. Configuration was never going to reach the real limitation.

**For subagents, an instruction is a suggestion; only the environment is a constraint.** I handed the upstream patch to a subagent in a split pane and told it, clearly, to push to a fork rather than upstream. Then I checked: the checkout's only remote was upstream itself, and no fork existed. The instruction was one confused moment away from a push to someone else's `master`. The fix is structural, not verbal:

```bash
gh repo fork owner/repo --remote --remote-name fork --clone=false
git remote add fork ssh://git@github.com/<you>/repo.git
git remote set-url --push origin DISABLED_push_to_fork_instead
```

```
fork    ssh://git@github.com/<you>/repo.git   (fetch/push)
origin  ssh://git@github.com/owner/repo.git   (fetch)
origin  DISABLED_push_to_fork_instead         (push)
```

An accidental `git push origin` now fails loudly instead of succeeding quietly. Worth telling the agent explicitly that this failure *is the guardrail working* — otherwise a helpful assistant will repair your safety net.

## Source

- [Herdr](https://herdr.dev) — the multiplexer, and its [repository](https://github.com/herdrdev/herdr)
- [WezTerm mouse binding reference](https://wezterm.org/config/mouse.html) — `mouse_reporting`, `bypass_mouse_reporting_modifiers`
- [Ghostty](https://ghostty.org) — whose VT engine provides the configurable link regexes
- [XTerm control sequences](https://invisible-island.net/xterm/ctlseqs/ctlseqs.html) — mouse tracking modes 1002 and 1006
