---
layout: post
title: "One registry, two modes: declarative remote-host access for you and your agent"
date: 2026-07-23 15:30:00 +0530
categories: ai agents productivity tools
series: "AI coding agent productivity"
---

You drive a handful of remote boxes from a Windows laptop — some over SSH, some over serial consoles on a console-server. Over months you accumulate a pile of glue: a `create-sessions` script for one lab, a hand-written `.cmd` wrapper for another host, a saved PuTTY session for a third, a `~/.ssh/config` block you tuned once and forgot. Each host was solved in isolation. Nothing knows about anything else. And when an AI agent joins the work, it has no idea any of it exists — so it fumbles, re-derives connection details, and trips the same host-key prompts you fixed by hand a year ago.

That fragmentation is the real problem. Not "which terminal" — but that the knowledge of *how to reach each box* is scattered across five places and reaches neither a new agent nor future-you. This post is about collapsing that into one declarative registry that generates everything downstream, and one design decision at its core: **two access modes, chosen deliberately.**

## The two modes (and why one echoes your command back)

There are two fundamentally different ways to run a command on a remote box, and they produce different output because they use different channels.

**Mode 1 — drive an interactive terminal window.** A real terminal is open (a PuTTY window, a `tmux`/`screen` pane) and you *type into it* — on Windows, often by posting `WM_CHAR` messages to the window. The remote shell runs under a pseudo-terminal (PTY), and **a PTY echoes back every character it receives**. So a captured transcript is literally what a human sees on screen:

```
whoami

root
root@web-01:~#
```

The command is *in* the output. So is the prompt.

**Mode 2 — one-off non-interactive exec.** You run `ssh web-01 whoami`. OpenSSH allocates no PTY, so there's no echo layer — you get exactly the command's stdout:

```
root
```

Same command, same host, clean — because there's no terminal in the loop to echo anything.

| | Interactive window | One-off exec |
|---|---|---|
| Invocation | type into an open PuTTY/`tmux` window | `ssh host "cmd"` |
| PTY allocated? | yes | no (unless you pass `-t`) |
| Command echoed in output? | **yes** | no |
| Prompt in output? | yes | no |
| Survives disconnect? | yes (window stays open) | no (process ends) |
| Good for | serial consoles, watching live, boot logs | scripting, parsing, clean capture |

### The mechanism: what a PTY actually does

A pseudo-terminal is a kernel device with two ends: the **master** (whatever drives — your terminal emulator, `tmux`, `expect`) and the **slave** (where the shell runs). The slave runs a *line discipline* — a small in-kernel state machine with an `ECHO` flag on by default. When the master writes bytes toward the shell, the line discipline echoes them back before the shell ever sees them. That's why you see your own keystrokes in any terminal — it isn't the shell printing them, it's the tty layer.

```bash
stty -a | grep -o '\-\?echo'   # 'echo' = on, '-echo' = off
ssh web-01 whoami              # no PTY  -> prints: root
ssh -tt web-01 whoami          # forces a PTY -> echoes the command + a prompt
```

Password prompts clear `ECHO` temporarily (`stty -echo`) so keystrokes don't show — same mechanism, deliberately off. With no PTY there's no slave-side line discipline at all, so nothing echoes. That single design fact is the whole reason the two modes differ — and the reason an agent must choose between them consciously.

## One registry, everything generated

The fix for the fragmentation is a single source of truth — a small YAML file listing every host — and one command that regenerates all the downstream artifacts from it. I call mine `pcon`. The registry looks like this:

```yaml
testbeds:
  web-01:
    kind: ssh
    host: 10.0.0.11
    user: root
    exec: web1          # short one-off wrapper name
    manage_ssh: true    # generate a ~/.ssh/config block if none exists
  console-a:
    kind: telnet        # serial console over a console-server port
    host: 10.0.0.254
    port: 2006
    titles: [console-a]
```

`pcon sync` reads that and regenerates, for each host:

- a **PuTTY saved session** (so a GUI session manager can open it),
- a **persistent-window wrapper** (`putty-web-01`) for the interactive mode,
- a **one-off wrapper** (`web1 <cmd>`) that runs `ssh web1 …` — the clean, no-PTY path,
- the **`~/.ssh/config` block** for that host.

Adding a box becomes one row plus `pcon sync` — not a new bespoke script. And crucially, the same registry is what you point an agent's onboarding docs at: one place, not five.

```bash
pcon list                 # every host + which windows are open
pcon add web-02 host=10.0.0.12 user=root exec=web2 manage_ssh=true
web2 uname -a             # one-off, clean output (no PTY, no echo)
pcon launch web-01        # persistent window you can see + scroll back
pcon rm web-02            # removes the row AND every generated artifact
```

## Design decisions that actually mattered

The tool is thin. The value is in a few decisions that came from *not* reinventing wheels:

- **Reuse existing primitives, don't rewrite them.** `pcon` orchestrates the session-creation, window-send, and theme scripts that already worked. It adds only the missing piece — the registry and the one-off wrappers — rather than a new monolith.
- **Never clobber hand-tuned config.** `sync` *reuses* an existing `~/.ssh/config` block if the host already has one (e.g. a `ProxyCommand` jump you tuned by hand); it only *generates* blocks for hosts that lack them, inside a delimited `# >>> managed >>>` region. `IdentityFile` is inherited from `Host *`, never duplicated.
- **`add` must preserve comments.** The first cut used PyYAML's `safe_dump`, which silently strips every comment and reorders keys — it parses into plain dicts and throws the token stream away. A self-documenting registry can't survive that. The fix is a round-trip parser (`ruamel.yaml`) that attaches comments to nodes and preserves them across load→modify→dump. **Any edit-in-place of a human-authored config file needs a round-trip parser.**
- **Lifecycle completeness.** `add` without `rm` accumulates orphans — dead wrappers, stale sessions, ssh blocks for hosts that no longer exist. That *is* the fragmentation you set out to kill. `rm` prunes all four artifact types.
- **Detached GUI launch.** Opening a window with a blocking `subprocess.call` hangs the shell until the window closes. A fire-and-forget GUI launch needs `DETACHED_PROCESS` (Windows) / a double-fork (Unix) so the command returns immediately.

On the "buy vs build" question: for the SSH side there *are* established declarative tools ([`assh`](https://github.com/moul/assh) compiles a YAML host file into `~/.ssh/config`, with gateway/`ProxyJump` chaining). For the GUI side, an established tabbed session manager beats hand-rolling one and imports the PuTTY sessions you already generate. The thin custom glue is worth writing only for the parts nothing off-the-shelf covers: the registry that unifies *both* modes plus serial consoles on Windows, and the one-off wrappers. Reach for the boring, established tool first; keep the custom surface as small as it can be.

## Why the two-mode split matters for the agent

- **Route anything the agent parses through one-off exec.** Clean stdout, an exit code, no prompt-scraping. If your agent greps command output, it should almost never read from an interactive window — the echoed command and prompt line are exactly the noise that breaks brittle parsing.
- **Use the interactive window only when there's no other channel.** Serial consoles (a device UART, a bootloader, a kernel before networking) have no sshd to `ssh host cmd` into — you must type into a session and read the screen, echo and all. It's also the mode a human wants for watching and scrollback.

The trap: standardising the agent on the interactive-window mode "because it's what I use" wraps every command in echo + prompt, and you end up writing fragile regexes to strip your own commands out. The fix isn't a better regex — it's routing parseable commands through the no-PTY path. When you *are* stuck in a window, a sentinel (`echo __BEGIN__; cmd; echo __END__`) and slicing between markers survives echo, prompts, and pagination far better than prompt-matching alone.

## The rule of thumb

> Put every host in one registry and generate the rest. If something reads the output, run it without a PTY (`ssh host cmd`). If a human — or a serial line — needs a live terminal, use the interactive window, and expect the echo.

The echoed command was never a bug in your tooling — it's the tty line discipline doing exactly what it's done since the 1970s. And the pile of per-host scripts was never a tooling problem either; it was a *source-of-truth* problem. Collapse it to one registry, pick the right side of the PTY boundary per command, and both you and your agent stop re-deriving what a file could have told them.

## Source

- PuTTY / plink (classic Windows terminal + scriptable link): [chiark.greenend.org.uk/~sgtatham/putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/)
- `assh` — declarative YAML → `~/.ssh/config` with gateways: [github.com/moul/assh](https://github.com/moul/assh)
- `ruamel.yaml` — round-trip YAML that preserves comments: [pypi.org/project/ruamel.yaml](https://pypi.org/project/ruamel.yaml/)
- OpenSSH `ssh` manual — `-T`/`-t` PTY allocation flags: [man.openbsd.org/ssh](https://man.openbsd.org/ssh)
- `termios` — pseudo-terminal line discipline and `ECHO`: [man7.org/linux/man-pages/man3/termios.3.html](https://man7.org/linux/man-pages/man3/termios.3.html)
- SSH Connection Protocol, pty-req (RFC 4254 §6.2): [rfc-editor.org/rfc/rfc4254](https://www.rfc-editor.org/rfc/rfc4254)
