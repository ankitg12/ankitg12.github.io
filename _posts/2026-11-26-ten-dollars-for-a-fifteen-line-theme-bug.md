---
layout: post
title: "$10.51 for a 15-Line Bug: Why Agentic Debugging Needs Pipeline Bisection Before 5 Whys"
date: 2026-11-26 07:30:00 +0530
categories: ai agents debugging
---

A terminal window stayed in dark mode after sunrise, even after a restart. Inside it, the [oh-my-pi](https://github.com/can1357/oh-my-pi) (`omp`) UI cards rendered in light mode while the [Tern](https://github.com/stencil-hq/tern-sdk) window chrome around them stayed dark. Every other desktop app already switched on schedule. Resolving it took one helper function in a Python theme script and one CLI command, yet [AgentsView](https://github.com/wesm/agentsview) tallied **$10.51 across two sessions** (`$7.38` on day one, `$3.14` on day two, 342 message rows).

The bug itself was small. The interesting story is *why* an AI coding agent spent ten dollars on it—and which debugging discipline cuts the same class of failure to under fifty cents.

## The actual bug (30 seconds once isolated)

On Linux with [GNOME](https://www.gnome.org/) and [Wayland](https://wayland.freedesktop.org/), a [systemd](https://systemd.io/) user timer switches themes at sunrise and sunset. Live probing of Tern `0.9.35` exposed three separate behaviors (filed as [stencil-hq/tern-sdk#437](https://github.com/stencil-hq/tern-sdk/issues/437)):

1. Tern reads the [XDG Desktop Portal](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.Settings.html) `org.freedesktop.appearance` `color-scheme` key only at startup; live `gsettings` changes do not update an open window.
2. `tern ctl appearance dark|light` returns `{"ok":true}` while leaving `tern ctl state`'s `.theme` unchanged.
3. Outside a Tern pane (such as a `systemd --user` service), `tern ctl` connects to `/tmp/tern.sock` and fails unless `TERN_PANE_SOCKET` points to `$XDG_RUNTIME_DIR/tern/daemon.sock`.

Writing `"theme": "Light"` or `"Dark"` into `~/.config/tern/settings.json` (while keeping `theme_light` and `theme_dark` unset so Tern reads both named palettes from `omp`'s `config.yml`) and invoking `reload_config` switches the open window silently with no popup and no restart:

```bash
TERN_PANE_SOCKET="/run/user/$(id -u)/tern/daemon.sock" \
  tern ctl plugins run reload_config
```

## Four agent failure modes that burned $10.51

| Failure mode | What happened | Cost impact |
|---|---|---|
| **Unverified note poisoning** | Day 1 wrote *"Tern follows GNOME `color-scheme`"* into local notes without running a live `dark -> light` switch on an open window. | Sent two later sessions chasing the wrong layer. |
| **Dirty interrupted experiment** | An evening test wrote `"theme": "Dark"` and `"theme_dark": "dark"` into `settings.json` and stopped mid-test without reverting. | Next morning, even restarting Tern stayed dark. |
| **Prose-first instead of probe-first** | Day 2 spent the opening minutes reading a 442-line notes monolith, SDK type definitions, and prior session transcripts before dumping live state. | Burned context tokens before checking a 16-line JSON file. |
| **Trusting `{"ok": true}` without verifying the consumer** | `tern ctl appearance dark` returned `{"ok": true}`, yet `tern ctl state` proved the rendered theme did not move. | Masked a no-op until checked directly against `tern ctl state`. |

## Why 5 Whys and Fishbone are the wrong tools *during* live debugging

When a bug burns ten dollars in agent tokens, the natural instinct is to ask whether [Five Whys](https://en.wikipedia.org/wiki/Five_whys) or an [Ishikawa (Fishbone) diagram](https://en.wikipedia.org/wiki/Ishikawa_diagram) would have prevented it.

Used *during* live debugging, both pull an LLM in the wrong direction: they reward prose speculation over measurement. Both belong **after** the bug is solved, as post-mortem tools for fixing the workflow.

During live debugging, two older engineering frameworks directly prevent token burn: [David Agans' *Debugging: The 9 Indispensable Rules*](https://debuggingrules.com/) and [Andreas Zeller's *Why Programs Fail*](https://www.whyprogramsfail.com/) (scientific debugging and pipeline bisection).

| Stage | Methodology | Concrete agent rule |
|---|---|---|
| **Minute 0 (live)** | Agans Rule 3 (*Quit Thinking and Look*) + Zeller pipeline bisection | List the hops (`timer -> script -> config/IPC -> consumer`) and dump every hop in **one** read-only command before reading long notes or past transcripts. |
| **Minute 1 (live)** | Agans Rule 7 (*Check the Plug*) | Inspect the live config on disk (`settings.json`) and test under `env -i` to reproduce minimal `systemd`/cron environments. |
| **During tests** | Agans Rules 5 & 6 (*Change One Thing* + *Audit Trail*) | Snapshot config before any exploratory edit and guard temporary changes with a shell `trap` restore so an interrupted turn never leaves a pinned config behind. |
| **Before "done"** | Agans Rule 2 (*Make It Fail*) | Prove a full live round-trip (`dark -> light` or `A -> B -> A`) against the consumer's state (`tern ctl state`), never just exit code `0` or `{"ok": true}`. |
| **After verified** | Toyoda's Five Whys / Ishikawa | Ask why diagnosis took more than one pipeline probe, then fix the durable artifact: split bloated notes into a router + command table, delete unverified claims, and file upstream bugs. |

## The single command that replaces two sessions

Theme sync is a four-hop pipeline: timer → script → desktop/config state → running window. Running one read-only snapshot at turn 1 isolates the broken hop in seconds:

```bash
# Hop 1-2: desktop and omp state
gsettings get org.gnome.desktop.interface color-scheme
gdbus call --session --dest org.freedesktop.portal.Desktop \
  --object-path /org/freedesktop/portal/desktop \
  --method org.freedesktop.portal.Settings.Read \
  org.freedesktop.appearance color-scheme
cat ~/.omp/agent/current-theme

# Hop 3-4: Tern disk config vs live window state
cat ~/.config/tern/settings.json
TERN_PANE_SOCKET="/run/user/$(id -u)/tern/daemon.sock" \
  tern ctl state | jq '{theme, theme_light, theme_dark}'
```

And the round-trip proof in a clean `systemd`-like environment before marking anything done:

```bash
env -i HOME="$HOME" USER="$USER" XDG_RUNTIME_DIR="/run/user/$(id -u)" \
  DBUS_SESSION_BUS_ADDRESS="$DBUS_SESSION_BUS_ADDRESS" \
  PATH="$HOME/.local/bin:/usr/bin:/bin" ~/.local/bin/tdark
tern ctl state | jq '{theme, theme_light, theme_dark}'

env -i HOME="$HOME" USER="$USER" XDG_RUNTIME_DIR="/run/user/$(id -u)" \
  DBUS_SESSION_BUS_ADDRESS="$DBUS_SESSION_BUS_ADDRESS" \
  PATH="$HOME/.local/bin:/usr/bin:/bin" ~/tools/linux/auto-theme.sh
tern ctl state | jq '{theme, theme_light, theme_dark}'
```

## Durable changes from the post-mortem

Running Five Whys *after* verification produced three concrete upgrades:

1. **Split the notes monolith:** Replaced a 442-line Tern notes file with a 24-line router plus five <50-line topic files, putting `tern ctl state` and `tern ctl plugins run reload_config` in one table.
2. **Filed the upstream bug:** [stencil-hq/tern-sdk#437](https://github.com/stencil-hq/tern-sdk/issues/437) documents the Linux portal, `tern ctl appearance`, and socket discovery gaps with minimal repros.
3. **Codified a `debug-it` skill:** An on-demand debugging checklist enforcing the four phases above (Turn-1 pipeline probe, rollback-guarded experiments, round-trip consumer verification, and post-fix Five Whys) with zero always-loaded prompt overhead.

## Source

- Upstream bug report: [stencil-hq/tern-sdk#437](https://github.com/stencil-hq/tern-sdk/issues/437)
- Debugging references: [David Agans — *Debugging: The 9 Indispensable Rules*](https://debuggingrules.com/), [Andreas Zeller — *Why Programs Fail*](https://www.whyprogramsfail.com/), [XDG Desktop Portal Settings API](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.Settings.html)
- Session cost telemetry: [AgentsView](https://github.com/wesm/agentsview)
