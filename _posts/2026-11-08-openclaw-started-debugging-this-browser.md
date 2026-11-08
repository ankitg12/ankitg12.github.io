---
layout: post
title: "\"OpenClaw\" Started Debugging This Browser: A Diwali Afternoon Banner"
date: 2026-11-08
categories: ai agents debugging
series: "AI coding agent productivity"
---

Edge put a bar under my address bar on Diwali afternoon: `"OpenClaw" started debugging this browser` with a `Cancel` button. I was on `news.ycombinator.com/newest`, doing nothing remotely debugging-shaped. It looked like an attack. It was my own relay.

This is the afternoon twin of this morning's [Diwali decorating post]({% post_url 2026-11-08-happy-diwali-zero-readers %}) — and the direct sequel to [the tab that was there but the read said wasn't]({% post_url 2026-11-02-tab-was-there-read-said-wasnt %}). Same relay, new symptom.

## The banner means one thing

Chromium shows that infobar for exactly one reason: an extension holds a `chrome.debugger` attachment on that tab. The [chrome.debugger API docs](https://developer.chrome.com/docs/extensions/reference/api/debugger) say it outright — attach, and the browser warns the human. No other OpenClaw path triggers it. The managed headless profile does not. The extension relay does.

The extension's own manifest says so:

```json
{
  "name": "OpenClaw",
  "permissions": ["debugger", "tabs", "tabGroups", "storage", "alarms", "nativeMessaging"]
}
```

Name matches the banner. Permission explains the banner.

## Two Edges, one screen

Two Edge processes were alive:

```bash
ps -p 4677 -o command=
# /Applications/Microsoft Edge.app/Contents/MacOS/Microsoft Edge

ps -p 56676 -o command=
# ... --remote-debugging-port=18800 --user-data-dir=~/.openclaw/browser/openclaw/user-data ...
```

The first is my daily Edge. The second is [OpenClaw](https://github.com/openclaw/openclaw)'s isolated automation browser on port `18800` with its own user-data dir. My screenshot was the first one — so the isolated browser was not the cause. The cause had to come from inside my own window.

`openclaw browser profiles` confirmed it:

```
google: stopped (port 18801)
openclaw: running, 0 tabs (port 18800)
user: stopped
chrome: running, 6 tabs [extension] (relayPort 18799)
```

The `chrome` profile is the extension relay. Its tab list was the giveaway:

```bash
openclaw browser tabs --browser-profile chrome
```

```
1. New Links | Hacker News — news.ycombinator.com/newest
2. torvalds/linux: Linux kernel source tree — github.com/torvalds/linux
...
6. New Links | Hacker News — news.ycombinator.com/newest
```

My exact URL, twice. The relay was attached to the tab I was reading.

## Why Cancel is safe

Clicking `Cancel` is a native detach. The extension listens for it:

```js
chromeApi.debugger.onDetach.addListener((source, reason) => {
  if (reason === "canceled_by_user") {
    // pause this tab: record revocation, then detach
    await pauseTab(source.tabId);
  }
  nativeDetached(source.tabId);
});
```

Simplified, but that is the shape: your click retires the debugger session and, in all-tabs mode, records a pause so the watchdog does not immediately reattach. The listening relay on `localhost:18799` stays up:

```
node 45606 TCP localhost:18799 (LISTEN)
```

`permissions_list_open` returned zero approvals. `events_poll` returned zero events. Nothing was waiting on the detach. The bar cleared and stayed cleared.

## Why not the heavier fixes

| Path | Result |
|---|---|
| `browser stop --browser-profile chrome` | Reported `running: true`, port `18799` still listening — relay is gateway-owned, stop does not kill it |
| Kill the gateway | Would drop WhatsApp + Telegram channels too; the banner is one tab, not the whole daemon |
| Disable the extension in Edge | Strongest, but loses the relay path that owns my logged-in sessions (same reason the managed profile could not substitute in the [last post]({% post_url 2026-11-02-tab-was-there-read-said-wasnt %})) |
| Click Cancel | Detaches one tab, records pause, banner gone |

The minimal action owns the symptom. The heavy actions own side effects.

## What I check first next time

1. Match the banner name to the extension manifest name before suspecting anything else.
2. `browser tabs --browser-profile chrome` — if my current URL is listed, the banner is mine.
3. Click Cancel, then check the port once: still listening is normal, reattached banner is not.

## Later

Whether per-tab pause survives an extension worker restart is untested — the MV3 watchdog reconnects every 30 seconds, and a restart could drop the pause list. Also open: a one-command `relay pause-all` so a festival afternoon never needs manifest archaeology. Neither blocks today. The bar is gone.

## Source

- [OpenClaw](https://github.com/openclaw/openclaw)
- [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/)
- [chrome.debugger API](https://developer.chrome.com/docs/extensions/reference/api/debugger)
