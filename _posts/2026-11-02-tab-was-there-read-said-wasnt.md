---
layout: post
title: "The Tab Was There. The Read Said It Wasn't."
date: 2026-11-02
categories: ai agents debugging
series: "AI coding agent productivity"
---

`tabs` listed two Google Activity pages. `snapshot --target-id t5` said tab not found. Same gateway, same second, same tab. That gap cost an hour.

The setup: [OpenClaw](https://github.com/openclaw/openclaw) drives a visible Edge window through a relay on `localhost:18799`, using the [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/) under the hood. Two gateway processes owned that relay port at once — the main app gateway and an orphaned relay daemon. Each `tabs` call alternated owners. Each owner kept its own tab-alias counter. So the first call returned `t5`, the next returned `t11`, then `t13`, then `t19`. Aliases rotated while raw target IDs stayed constant. A snapshot issued with a stale alias hit the wrong owner and failed.

Killing the orphan (`kill 5807`, one owner left on `:18799`) stopped the flapping. But a second failure remained: aliases are short-lived per CLI invocation. The fix is what I call an **atomic read** — fetch `tabs --json`, extract the raw `targetId`, and issue the read with that ID in the same shell chain before anything else re-lists:

```bash
TABJSON=$(openclaw browser --browser-profile chrome tabs --json)
TID=$(echo "$TABJSON" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['tabs'][0]['targetId'])")
openclaw browser --browser-profile chrome snapshot --target-id "$TID" --limit 1000
```

Raw IDs (`7AF5D6C6...`) survive across owners. Aliases (`t5`) do not. The read that failed for an hour succeeded on the first atomic attempt.

## Why not just use the managed profile?

OpenClaw also runs a headless Edge on port `18801` with a stable `activity` label. That tab is rock-solid — but it has no Google session. Cookie sync failed with a locked database, and direct history reads hit macOS TCC. The visible Edge had the login; the relay to it was the flaky path. Atomic reads fix the flaky path without moving the session.

## Later

Alias churn per CLI call looks like a gateway restart resetting a counter. Worth confirming whether persistent gateway mode keeps aliases stable. Also: daylog builder misses `__mba`-suffixed journals, so weekend sessions report zero — separate fix.

## Source

- [OpenClaw](https://github.com/openclaw/openclaw)
- [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/)
