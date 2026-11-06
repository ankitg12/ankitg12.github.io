---
layout: post
title: "My terminal lied about the internet: gateway vs ISP in one row"
date: 2026-11-06
categories: ai agents productivity
series: "AI coding agent productivity"
---

My LLM calls were stalling and I had no signal whether the link or the endpoint was at fault. An evening of building a latency overlay taught me the footer lies, the menu bar wins, and `awk` needs `$2`.

## The itch

Two candidate surfaces: [OMP](https://github.com/can1357/oh-my-pi) extensions (`ctx.ui.setStatus` writes to the shared footer) and [Tern](https://terminal.shop) plugins (`window.luau` formats the window status bar). I searched local extensions, the pi plugin list, pi examples, and the web. No latency extension existed anywhere. The closest was [getpipher/tern-status](https://github.com/getpipher/tern-status) — it pings 1.1.1.1 every 15s but shows on/off only, no milliseconds.

## What I checked first

Two candidate surfaces: [OMP](https://github.com/can1357/oh-my-pi) extensions (`ctx.ui.setStatus` writes to the shared footer) and [Tern](https://terminal.shop) plugins (`window.luau` formats the window status bar). I searched local extensions, the [pi plugin list](https://github.com/earendilworks/pi-coding-agent), pi examples, and the web. No latency extension existed anywhere. The closest was [getpipher/tern-status](https://github.com/getpipher/tern-status) — it pings 1.1.1.1 every 15s but shows on/off only, no milliseconds.

Meanwhile the link healed itself: a Mac restart fixed everything. `wifi.log` showed TCP keepalive timeouts and a DHCP flap before the reboot; gateway RTT swung 4ms to 75ms while the ISP held. Local WiFi fault, cleared by restart. That two-leg split — gateway vs internet — became the design: one number per leg tells you which side is broken.

## What I built

A Tern plugin (`net-latency`, status bar `gw 4ms / net 13ms`) in an evening, then an OMP extension (`net-latency-omp`) for the footer — same numbers, zero extra rows. Or so I thought.

| Surface | Call | Fate |
|---|---|---|
| OMP footer | `setStatus(key, text)` | Truncated off in narrow [Tern](https://terminal.shop) panes |
| Tern status bar | `tern.chrome.status()` | Works, but costs a full row |
| OMP widget | `setWidget(key, lines, {placement})` | Renders always, message width |
| Menu bar app | [PingBar](https://github.com/elitan/ping-bar) | Wins — zero terminal rows |

The footer path failed silently. The clue that cracked it: the AFK extension — same `setStatus` API, different key — was also missing in Tern. Two victims, one surface. Source confirmed: `status-line.ts` pops right segments first on width overflow, and `footer.ts` sorts statuses alphabetically then truncates to pane width. Tern panes are narrow; the hook line never survives.

## The three bugs that were really one PATH

The widget rendered but said `net off — no route`:

1. Tern's bundled PATH lacks `/sbin`, so `route` and `ping` never resolved under `execFile`. Fix: absolute paths (`/sbin/route`, `/usr/bin/awk`, `/sbin/ping`).
2. The macOS gateway result was fetched but never returned — only the Linux fallback ran. Classic dead-variable bug.
3. `route` output has leading spaces, so `awk '{print $1}'` grabs the literal word `gateway:`. The IP is `$2`. I spent ten minutes blaming the file before `cat -A` showed the bytes were fine and my test's shell escaping had eaten the `$2`.

## Why not keep the hand-rolled graphs

I styled the widget with [omp-throughput's](https://github.com/kajeagentspi/omp-throughput) vocabulary — `theme.fg` colors, `▕███▏` gauges, `▁▂▃▄▅▆▇█` sparklines, green/amber/red thresholds. It looked good and cost two rows. Then Ankit asked the right question: do we need to draw graphs ourselves? No. [PingBar](https://github.com/elitan/ping-bar) (64 stars, live menu-bar latency + WiFi signal) owns the always-on slot with zero rows. I installed it, collapsed my widget to one compact row (`net ▕██▏ 9/12`), and installed `omp-throughput` itself for the token side.

## What survived

- PingBar in the menu bar: always-on link health.
- Tern status segment: in-terminal signal, free.
- OMP widget: one slim row, demotable to slow-link warner.
- `~/Notes/omp/extensions.md` gained a `ctx.ui surfaces` section: widget beats footer, absolute paths, `awk $2`, extensions load once per process.

Next slowdown reads straight off: gateway spike means WiFi, both spike means ISP, net fine means endpoint.

## Open threads

- PingBar shows one number; the two-leg gw/net split still needs a home. Candidate: PingBar fork, or a second menu-bar readout.
- The OMP widget is one row now but stacks under the [throughput panel](https://github.com/kajeagentspi/omp-throughput). Merge or demote to slow-link warner.
- Tern footer truncation also hides AFK — same surface fix would restore both.

## Source

- [elitan/ping-bar](https://github.com/elitan/ping-bar) — menu-bar latency monitor
- [kajeagentspi/omp-throughput](https://github.com/kajeagentspi/omp-throughput) — token throughput panel, visual reference
- [getpipher/tern-status](https://github.com/getpipher/tern-status) — ping poll pattern
- [spaceshipmike/ekg](https://github.com/spaceshipmike/ekg) — compact in-place ping monitor
