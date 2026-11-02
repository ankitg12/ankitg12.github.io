---
layout: post
title: "The Tab Was There. The Read Said It Wasn't."
date: 2026-11-02
categories: ai agents debugging
series: "AI coding agent productivity"
---

`tabs` listed two Google Activity pages. `snapshot --target-id t5` said tab not found. Same gateway, same second, same tab. That gap cost an hour of retries before I stopped retrying and read the source.

## The stack

[OpenClaw](https://github.com/openclaw/openclaw) drives browsers two ways. A managed headless Edge over CDP on port `18801`, and a visible browser through an extension relay on `localhost:18799`. The relay bridges Playwright's CDP endpoint to the extension's `chrome.debugger` transport. The [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/) sits underneath both. One class owns the relay side:

```js
// relay-server — one bridge per extension-driver profile
var ExtensionRelayBridge = class {
  // Accepts at most one extension connection
  // (a newer one replaces the old — MV3 workers restart freely)
  // and any number of CDP clients.
}
```

That comment is the whole bug in one sentence. The bridge accepts **at most one** extension connection. A newer one replaces the old. No error, no warning to the evicted side. The old socket just stops getting answers.

## Two owners, one port

`lsof -i :18799` showed two listeners fighting over the relay:

```
bun  3903  TCP localhost:18799 (LISTEN)       # main app gateway
bun  5807  TCP localhost:18799->... (PAIR)     # orphan relay daemon
```

PID 3903 was the app gateway. PID 5807 was an orphaned relay daemon from an earlier session that never exited. Each CLI invocation connected to whichever owner answered first. Tab listing worked on both — both could see the Edge tabs. But tab **aliases** are per-process state:

```js
// target-id.ts — alias counter lives in profile state, in memory
function getTabAliasState(profileState) {
  profileState.tabAliases ??= { nextTabNumber: 1, byTargetId: {} };
  return profileState.tabAliases;
}
function assignTabAlias(params) {
  let entry = aliases.byTargetId[params.tab.targetId];
  if (!entry) {
    entry = { tabId: `t${aliases.nextTabNumber}` };
    aliases.nextTabNumber += 1;
    // ...
  }
}
```

`nextTabNumber` starts at 1 in each process and increments independently. Owner A calls the first Activity tab `t5`. Owner B calls it `t11`, then `t13`, then `t19` as its own counter advances. Raw CDP `targetId` values (`7AF5D6C6...`, `684A649D...`) are stable — they come from the browser. Aliases are not — they come from whichever gateway answered last.

So the observed sequence — `t5` not found, `t6` not found, `t11`, `t13`, `t19`, each valid for exactly one call — was two counters leapfrogging each other. A snapshot issued with owner A's alias landed on owner B, which had never assigned that alias. Hence `tab not found`.

The fix for this half is one line:

```bash
kill 5807   # orphan relay daemon; single owner left on :18799
```

After that, `tabs` returned stable aliases (`t19`, `t20`) because only one counter existed.

## Aliases are still short-lived

Single owner or not, aliases are assigned per CLI invocation against in-memory profile state. A separate `tabs` call minutes later can renumber. The durable fix is what I call an **atomic read**: fetch `tabs --json`, extract the raw `targetId`, and issue the read in the same shell chain so no re-list intervenes:

```bash
TABJSON=$(openclaw browser --browser-profile chrome tabs --json)
TID=$(echo "$TABJSON" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['tabs'][0]['targetId'])")
openclaw browser --browser-profile chrome snapshot --target-id "$TID" --limit 1000
```

Raw IDs survive across owners, restarts, and renumbers. Aliases survive one call. The read that failed for an hour succeeded on the first atomic attempt, returning the full signed-in Activity page — account header, controls, and today's entries in one snapshot.

## Why not just use the managed profile?

OpenClaw also runs headless Edge on `:18801` with a stable `activity` label. Rock-solid tabs, zero flapping. But it has no Google session. Three alternatives failed:

| Path | Result |
|---|---|
| Cookie sync from Chrome | `Error: unable to open database file` (locked while Chrome runs) |
| Direct history file copy | `Operation not permitted` (macOS TCC) |
| Safari history query | `authorization denied` (same TCC wall) |

The visible Edge had the login; the relay to it was the flaky path. Atomic reads fix the flaky path without moving the session anywhere.

## What I would check next time first

1. `lsof -i :18799` before the third tab retry. Two owners means kill the orphan, not retry the alias.
2. `tabs --json` over plain `tabs`. Raw `targetId` is the stable key; `tN` is a hint.
3. Same-shell `tabs` → read chaining for any extension-relay profile, always.

## Later

Whether persistent gateway mode (long-lived process, no per-call restart) keeps aliases stable across minutes is untested. Also unrelated but found during the same session: the daylog builder misses `__mba`-suffixed journals, so weekend sessions report zero despite full logs — separate fix queued.

## Source

- [OpenClaw](https://github.com/openclaw/openclaw)
- [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/)
