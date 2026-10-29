---
layout: post
title: "^[[200~ on the First Paste: An Unfinished Investigation"
date: 2026-10-29
categories: linux terminal debugging
series: "Moving to Linux"
---

I pasted a command into [Ghostty](https://ghostty.org/) on [Ubuntu](https://ubuntu.com/) and the prompt showed `^[[200~` before the text. The shell coloured the whole line red as a syntax error. I had seen the same thing in a second terminal emulator.

The fault is still not fixed. This post records what is known, what was tested, and what to capture the next time it happens.

## What the characters are

`ESC [ 200 ~` is the start marker of **bracketed paste**, private mode 2004 in the [xterm control sequences](https://invisible-island.net/xterm/ctlseqs/ctlseqs.html). When an application turns the mode on with `ESC[?2004h`, the terminal wraps every paste:

```
ESC[200~ <pasted text> ESC[201~
```

The shell then knows that the text came from the clipboard and not from the keyboard. So a pasted newline does not run the command. If the program that receives the paste does not handle the markers at that moment, the markers show as text. `^[` is how the terminal shows `ESC`.

[Bash](https://www.gnu.org/software/bash/) ([readline](https://tiswww.case.edu/php/chet/readline/rltop.html)), [ble.sh](https://github.com/akinomyoga/ble.sh), vim and most TUIs turn the mode on. Background: [cirw.in, "bracketed paste mode"](https://cirw.in/blog/bracketed-paste) and [Wikipedia: Bracketed-paste](https://en.wikipedia.org/wiki/Bracketed-paste).

## This is a known symptom

A search finds the problem quickly, including the word "sometimes":

- [Ask Ubuntu: "When I paste in the terminal sometimes the contents are prefixed with the characters '^[[200~'"](https://askubuntu.com/questions/1396132/when-i-paste-in-the-terminal-sometimes-the-contents-are-prefixed-with-the-charac)
- [Unix & Linux SE: "Bracketed Paste Mode gone wrong"](https://unix.stackexchange.com/questions/754302/bracketed-paste-mode-gone-wrong)
- [Guake issue #947](https://github.com/Guake/guake/issues/947): the markers trigger the readline numeric argument prompt in vi-mode.

The usual causes given there:

1. A TUI exits or crashes and does not turn mode 2004 off.
2. A program that reads raw input is in the foreground.
3. A layer between the terminal and the shell (multiplexer, ssh, nested shell) wraps the paste a second time.

The usual workaround: press Ctrl+C and paste again, or run `reset`.

## What I asked the agent, and what it checked

I asked my coding agent why it happens. It spent about ten minutes and about $1.10 of model usage on these checks.

| Check | Result |
|---|---|
| ESC bytes in the clipboard (`wl-paste -n \| tr -cd '\033' \| wc -c`) | `0`: the clipboard holds plain text |
| Line editor on the prompt | ble.sh `0.4.0-devel4+d81fd54`, not plain readline |
| Earlier occurrences in `~/.bash_history` | one stored line starts with `ESC[200~` before an `npm install` command |
| Plain readline (`bash --norc -i`) in a test pty, paste wrapped twice | `literal 200~: True`: the inner marker shows as text |
| Plain readline: paste right after `cd`; ESC byte 60 ms before the rest | clean |
| ble.sh in a test pty, `TERM=xterm-ghostty` and `xterm-kitty`: single paste, double-wrapped paste, ESC 0.6 s early, paste during `cd` | clean, all 8 cases |
| ble.sh: Esc, Alt, Ctrl+V or Ctrl+Q pressed before the paste | no `^[[200~` (Ctrl+V inserted `[200~` without the ESC) |
| ble.sh: focus-out/focus-in report, cursor-position report or DA2 reply arriving next to the paste | clean |
| ble.sh: paste body 0.05 s, 0.3 s or 1.0 s after the start marker; marker split inside the CSI | clean |

The test pty was a Python `pty.fork()` that started `bash -i` and wrote the bytes a terminal would send. It does not run the real terminal, the real clipboard or a long-lived shell.

## Two capture attempts that recorded nothing

The next step was to record the bytes that the real terminal sends. The agent started a new Ghostty window with [`script`](https://man7.org/linux/man-pages/man1/script.1.html) logging input:

```bash
setsid -f ghostty --title=paste-capture -e \
  script -q --log-in /tmp/paste-in.log --log-timing /tmp/paste-in.tm -c 'bash -i'
```

The log on the host stayed at 0 bytes. This Ghostty is the [snap](https://snapcraft.io/ghostty) build. `pgrep` showed it running under `bwrap` ([bubblewrap](https://github.com/containers/bubblewrap)), so its `/tmp` is not the host `/tmp`.

The second attempt wrote to `$HOME` instead. That log also stayed at 0 bytes, also after a `SIGTERM` to `script`. Both times my pastes in the capture window were clean.

## What I saw myself

- The fault is on the **first** paste after a new copy. A second Ctrl+Shift+V of the same text is clean.
- A later copy from [Zed](https://zed.dev/), pasted the same way, was clean.

So the fault does not happen on every paste. The agent could not reproduce it during the session.

## The history file kept the evidence

After I closed the Ghostty tab of the screenshot, bash wrote that tab's history. Only two lines in `~/.bash_history` contain an ESC byte. `cat -v` shows each one with the line after it:

```
^[[200~npm install -g @github/copilot
npm install -g @github/copilot
...
^[[200~<the command from the screenshot>
<the command from the screenshot>
```

`od -c` of the second bad line starts with `033 [ 2 0 0 ~` and ends with `g e m i n i \n`. The pattern is the same both times:

- The start marker `ESC[200~` is stored as text, with a real ESC byte.
- The end marker `ESC[201~` is not stored.
- The next history line is the same command without the marker: the second paste was clean.

So the shell did recognise the end marker (or it never arrived) but took the start marker as typed text. If bracketed paste were off, there would be no markers. If the line editor did not handle the markers at all, both markers would show as text.

[INFERENCE] On the first paste, the start marker reached the line editor in a form or at a time where the decoder did not take it as one key sequence. To confirm or refute this, I need a timed byte capture of a failing first paste. The synthetic tests above sent the same bytes and stayed clean, so the difference is in what the real terminal sends or when.

## Google or the agent?

During the session I asked: am I wrong to use an LLM where a Google search could have been enough?

For "what are these characters, and how do I clear them", the search is enough. The first result names the symptom, and the workaround is one line. For "why does it happen on my machine", a search cannot answer, and the agent could not answer either without a reproduction. Each extra synthetic test made the guess narrower but proved nothing.

The order I would use now:

1. Search for the symptom first.
2. Use the agent only for the local cause, and only when the fault can be made to occur, or when there is a byte capture of a failure.

## Not checked

- Why the line editor took the start marker as text but not the end marker.
- Whether a TUI that exited in that tab left the terminal in a bad state before the paste.
- Whether the copy source (terminal selection, browser, editor) makes a difference.
- Why the second `script` capture, written to `$HOME`, stayed empty.
- Whether the second terminal emulator fails in the same way.

## Next time it happens

The session ended with this checklist:

1. Write down what that tab did just before the paste: a new tab, a TUI exit, Ctrl+C on a command, a long idle time.
2. Note where the text was copied from.
3. Get a timed byte capture of a failing first paste from a terminal that is not sandboxed, so the log is visible on the host.

Status: open, no reproduction yet.

## Source

- [XTerm Control Sequences: bracketed paste mode](https://invisible-island.net/xterm/ctlseqs/ctlseqs.html)
- [ble.sh](https://github.com/akinomyoga/ble.sh)
- [cirw.in: bracketed paste mode](https://cirw.in/blog/bracketed-paste)
- [Ask Ubuntu 1396132](https://askubuntu.com/questions/1396132/when-i-paste-in-the-terminal-sometimes-the-contents-are-prefixed-with-the-charac)
- [Unix & Linux SE 754302](https://unix.stackexchange.com/questions/754302/bracketed-paste-mode-gone-wrong)
- [Guake issue #947](https://github.com/Guake/guake/issues/947)
