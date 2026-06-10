---
layout: post
title: "Making terminal URLs clickable in an AI coding agent — four approaches, one that works"
date: 2026-06-10
categories: ai tools productivity windows
series: "AI coding agent productivity"
---

I run a web search tool inside OMP (Oh My Pi) — my AI coding agent. It queries DuckDuckGo and prints results to the terminal. The problem: long URLs get truncated in the agent's output, and even the ones that aren't truncated aren't always clickable.

One hour later I had a local URL shortener backed by SQLite. Here's the path.

---

## The setup

My `search.py` tool calls ddgr, parses results, and prints title + URL. I invoke it through OMP's bash tool. The output renders in OMP's tool result pane inside WezTerm.

WezTerm auto-detects `https://` URLs in terminal text and makes them clickable — as long as the URL fits on one line.

The constraint: **OMP's bash tool output isn't rendered by WezTerm directly.** It goes through OMP's `OutputSink`, which captures stdout as text and renders it in its own UI component. Anything that requires the terminal parser — like OSC 8 hyperlink sequences — has to survive that pipeline.

Mostly, it doesn't.

---

## Approach 1: Rich `[link=URL]` markup

[Rich](https://github.com/Textualize/rich) supports OSC 8 hyperlinks via its `[link=URL]text[/link]` markup. Or so I thought.

```python
from rich.console import Console
console = Console(highlight=False)
console.print(f"[cyan][link={url}]{url}[/link][/cyan]")
```

Testing it:

```python
from rich.console import Console
from io import StringIO
c = Console(file=StringIO(), highlight=False)
c.print("[cyan][link=https://example.com]example.com[/link][/cyan]")
print(repr(c.file.getvalue()))
# 'example.com\n'
```

Rich strips OSC 8 on Windows. No escape sequences. Just text.

Even with `force_terminal=True`, the output was `example.com^M`. No OSC 8 bytes anywhere.

---

## Approach 2: Raw OSC 8 escape sequences

Skip Rich. Emit the bytes directly.

OSC 8 format: `\033]8;;URL\033\\ display text \033]8;;\033\\`

```python
OSC8 = "\033]8;;"
ST   = "\033\\"
print(f"   {OSC8}{url}{ST}{display}{OSC8}{ST}")
```

This should work — we're bypassing Rich entirely. The raw bytes go to stdout, OMP captures them, renders the text to WezTerm's PTY, WezTerm parses OSC 8, done.

Except OMP's non-PTY bash path uses `OutputSink` which sanitises output via `sanitizeText` before rendering. The OSC 8 bytes are stripped before they reach WezTerm's parser.

There's an existing OMP issue for this: [#1244](https://github.com/can1357/oh-my-pi/issues/1244), closed via PR #1248 — but that PR added OSC 8 support only for OMP's own tool renderers (`read`, `find`, `edit`, `search`, `ast_grep`). File paths, not arbitrary stdout from bash commands.

I filed [oh-my-pi#2221](https://github.com/can1357/oh-my-pi/issues/2221) for the bash output case.

---

## Approach 3: Full URL on its own line

If escape sequences don't survive, lean on WezTerm's regex URL detection. It matches `\bhttps?://\S+\b` in rendered text. If the full URL is on its own line and fits within the terminal width (~120 chars), WezTerm makes it clickable automatically.

```python
console.print(f"[bold]{title}[/bold]")
print(f"   \033[36m{url}\033[0m")
```

This works for short URLs. For long ones — TI SDK docs, deep GitHub links, anything with a query string — the URL wraps at column 120. WezTerm's regex can't match a URL that wraps across lines. Not clickable.

---

## The actual fix: local redirect server

The constraint is clear: the output pipeline won't carry OSC 8, and long URLs wrap. The only way to guarantee a clickable link in OMP output is to make the URL short enough that it never wraps.

`http://localhost:9999/a3f2b1` — 30 characters. Always fits. WezTerm auto-detects it. One click opens Edge.

Two files:

**`url_server.py`** — tiny HTTP server, reads a SQLite database, 302-redirects `/<hash>` to the full URL:

```python
class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        path = self.path.lstrip("/")
        with _db() as con:
            row = con.execute("SELECT url FROM urls WHERE id = ?", (path,)).fetchone()
        if row:
            self.send_response(302)
            self.send_header("Location", row[0])
            self.end_headers()
```

**`search.py`** — for URLs over 120 chars, compute a 6-char SHA256 hash, store in SQLite, display the short link. Short URLs pass through unchanged.

```python
def _shorten(url: str, title: str) -> str:
    h = hashlib.sha256(url.encode()).hexdigest()[:6]
    with _db() as con:
        con.execute(
            "INSERT OR IGNORE INTO urls(id, url, title) VALUES (?, ?, ?)",
            (h, url, title),
        )
    return h

# in the output loop:
if len(url) > 120:
    code  = _shorten(url, title)
    short = f"http://localhost:{PORT}/{code}"
    console.print(f"[bold]{title}[/bold]")
    console.print(f"   [cyan]{short}[/cyan]")
    console.print(f"   [dim]{url}[/dim]")   # visible, not clickable, just for context
else:
    console.print(f"[bold]{title}[/bold]")
    console.print(f"   [cyan]{url}[/cyan]")
```

The server starts as a detached daemon on first search, stays running across sessions. Same URL always gets the same hash — no duplicates, no stale entries.

Output looks like:

```
3.2.2.13. OSPI/QSPI NOR/NAND — Processor SDK Linux for AM57X Documentation
   http://localhost:9999/663b79
   https://software-dl.ti.com/processor-sdk-linux/.../QSPI.html

LKML: [PATCH] spi: cadence-quadspi: fix write completion support
   https://lkml.org/lkml/2022/4/29/432
```

Short links are clickable. Long URLs are readable. Short URLs pass through and are also clickable.

---

## What I learned

**`$WEZTERM_PANE`** is the canonical runtime check for WezTerm. OMP uses it in `isHyperlinkEnabled()`. Any script that conditionally emits OSC 8 should gate on this:

```python
import os
osc8_ok = bool(os.environ.get("WEZTERM_PANE"))
```

**`INSERT OR IGNORE` + hash id** is the right dedup pattern for single-writer tools. No transactions, no race conditions, deterministic short codes.

**The render path is the constraint.** Once you know output goes through `OutputSink` → sanitise → text render, you stop fighting escape sequences and start working with what the render path supports (plain URLs on their own line).

---

Code: [gist.github.com/ankitg12/f3e9f955455fc303553ce51f54ebf658](https://gist.github.com/ankitg12/f3e9f955455fc303553ce51f54ebf658)

OMP issue filed for upstream fix: [oh-my-pi#2221](https://github.com/can1357/oh-my-pi/issues/2221)
