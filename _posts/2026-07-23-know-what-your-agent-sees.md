---
layout: post
title: "You Can Only Optimize What You Can See: Measuring an Agent's Context Footprint"
date: 2026-07-23 14:00:00 +0530
categories: ai agents productivity tools
series: "AI coding agent productivity"
---

You disable a few MCP servers, trim some skills, and the agent *feels* snappier. But you're guessing. How many tokens does your coding agent actually load before it reads a single line of your question — and which part of that is even optimizable? If you can't answer in numbers, every "cleanup" is superstition.

This post is about turning that guess into two numbers you can watch move.

## The two numbers that matter

Every turn, an agentic harness sends the model a single flat prompt. It has two halves with completely different optimization levers:

| Half | What it is | Grows with | Lever |
|---|---|---|---|
| **Static** | System prompt + tool schemas + injected long-term memories | Your config | Fewer skills, fewer tools, memory curation |
| **Dynamic** | Conversation history + every tool result so far | Time-in-session | Checkpoint/rewind, compaction, tighter reads |

The static half is fixed the moment the session boots. The dynamic half only grows. Conflating them is why context optimization feels like whack-a-mole — you tune skills (static) while the thing eating your window mid-session is tool output (dynamic).

The trick is measuring each independently. I do this against [Oh My Pi](https://github.com/can1357/oh-my-pi) (OMP), whose RPC mode and session logs expose exactly the right numbers, but the *technique* generalizes to any harness that (a) can dump its rendered prompt and (b) logs per-turn token accounting.

## Number 1: the static footprint (a fresh session)

Don't ask the model "what skills do you have loaded" — that's a tool call to answer a question the harness already knows the deterministic answer to, and it hallucinates. Ask the harness.

OMP has a headless RPC mode. Send it a `get_state` request and it returns the fully rendered system prompt plus its own token accounting:

```bash
cd /path/to/agent-project      # cwd matters — project config applies here
printf '{"id":"1","type":"get_state"}\n' | omp --mode rpc --no-session
```

The reply is one JSON line. The fields you want:

- `data.contextUsage.tokens` — the harness's authoritative static token count.
- `data.systemPrompt` — the actual rendered text (so you can `grep` the `<skills>` block).
- `data.dumpTools` — the tool schemas, which also count toward the footprint.

A minimal extractor:

```python
import json, subprocess

def static_footprint(cwd):
    proc = subprocess.run(
        ["omp", "--mode", "rpc", "--no-session"],
        input='{"id":"1","type":"get_state"}\n',
        capture_output=True, text=True,
        encoding="utf-8", errors="replace",   # see the gotcha at the end
        cwd=cwd, timeout=120,
    )
    for line in proc.stdout.splitlines():
        try: d = json.loads(line)
        except json.JSONDecodeError: continue
        if d.get("type") == "response" and d.get("command") == "get_state":
            data = d["data"]
            return data["contextUsage"]["tokens"], data["systemPrompt"]
```

For one of my agents this printed **~31,000 tokens** — 105 skills, 13 tool schemas, ~84k characters of prompt. Now I know the number. And critically: **the system prompt is rebuilt at every launch and is never persisted to the session log**, so RPC is the *only* grounded way to read it. Grepping an old session file won't work.

## The static lever: ignore skills you never use

In OMP, skills are markdown capability files injected into the system prompt. A general install carries every skill for every context. A QA agent doesn't need the blog-publishing skill; a writing agent doesn't need the firmware-testbed skill. Each one is dead weight in that agent's prompt.

The fix is a per-project config that filters the skill list by glob on the skill name:

```yaml
# <project>/.omp/config.yml
skills:
  ignoredSkills:
    - "outlook-*"
    - "some-other-project-*"
    - "personal-*"
```

Across a fleet of five specialized agents, mine were each loading ~150 skills — most irrelevant to that agent's job. Filtering the wrong-domain families dropped each agent's skill count by 20–30% with zero capability loss (an ignored skill stays reachable on demand; it just isn't *pre-loaded* into every prompt). The `contextUsage.tokens` number is how you verify the cut actually landed instead of trusting the YAML.

## Number 2: the live footprint (a running session)

The fresh-session number is only half the story. Ten turns into real work, your context is dominated by history. To see *that*, read the session log.

OMP writes a JSONL file per session. Every assistant message carries a `contextSnapshot`:

```json
{ "promptTokens": 122442, "nonMessageTokens": 32538 }
```

- `promptTokens` — the full context sent that turn (equals `usage.input + cacheRead + cacheWrite`).
- `nonMessageTokens` — the static subset (prompt + tools + memories).

The decomposition falls right out:

```
static  = nonMessageTokens
dynamic = promptTokens - nonMessageTokens
```

Read the last assistant snapshot from the newest log and you have a live split:

```python
import json
from pathlib import Path

def live_split(session_jsonl):
    snap = None
    for line in Path(session_jsonl).read_text(encoding="utf-8").splitlines():
        if '"contextSnapshot"' not in line:
            continue
        msg = json.loads(line).get("message", {})
        if "contextSnapshot" in msg:
            snap = msg["contextSnapshot"]          # keep the last one
    total = snap["promptTokens"]
    static = snap["nonMessageTokens"]
    return static, total - static                  # (static, dynamic)
```

Watching a real session, the number that stopped me:

| | Live session, mid-work |
|---|---|
| Total loaded | ~124,000 tokens |
| Static (config) | ~32,500 (**26%**) |
| Dynamic (history + tool output) | ~92,000 (**74%**) |

**Three-quarters of the context was conversation, not configuration.** The skill-trimming was real and worth doing — but it optimizes the 26%. Mid-session, the lever that matters is discipline on the dynamic half: checkpoint/rewind around exploration, compact when it bloats, and stop dumping 500-line files into context when a 20-line slice would do.

Two footnotes from actually building this:

- **The log lags the live session.** Recent turns buffer in memory before they're flushed to disk, so the "last snapshot" can trail wall-clock by a minute or two. Print the turn's timestamp so you're never fooled into reading a stale number as current.
- **Timestamps are UTC.** I briefly thought my reader was pulling a 5-hour-old snapshot. It wasn't — the log was UTC and my shell clock was UTC+5:30. Same instant. Always compare in one timezone.

## The gotcha that cost me a bug report

I shipped the tool after testing it in a UTF-8 shell. It crashed instantly for the person running it in PowerShell:

```
UnicodeDecodeError: 'charmap' codec can't decode byte 0x8d
AttributeError: 'NoneType' object has no attribute 'splitlines'
```

Root cause: Python's `subprocess.run(..., text=True)` decodes the child's output using `locale.getpreferredencoding()`, which on Windows is **cp1252**, *regardless of what the child actually emits*. The RPC output was UTF-8; the first non-Latin-1 byte killed the reader thread, `stdout` came back `None`, and the next line blew up. The same trap bites a second time on the *output* side — printing box-drawing characters or non-ASCII to a cp1252 stdout throws too.

The fix is to never trust the platform default:

```python
# decoding the subprocess
subprocess.run(..., text=True, encoding="utf-8", errors="replace")

# and your own stdout/stderr
for stream in (sys.stdout, sys.stderr):
    if hasattr(stream, "reconfigure"):
        stream.reconfigure(encoding="utf-8", errors="replace")
```

The real lesson wasn't the encoding — it's that **a CLI tool tested only in your own shell is untested.** Cross-platform tools need to run in the consumer's actual environment before you call them done.

## Takeaways

- An agent's context has a **static** half (config, fixed at boot) and a **dynamic** half (history, grows over time). They have different levers; measure them separately.
- Get the static number from the **harness itself** (OMP: RPC `get_state` → `contextUsage.tokens`), not from asking the model.
- Get the live split from the **session log** (`contextSnapshot`: `promptTokens` vs `nonMessageTokens`).
- Trim static with per-project skill/tool filtering; verify the cut with the token number.
- Mid-session, the dynamic half usually dominates — so checkpoint/rewind and compaction matter more than shaving the prompt.
- On Windows, always pin `encoding="utf-8"` on both subprocess I/O and your own stdout.

## Source

- [Oh My Pi (OMP)](https://github.com/can1357/oh-my-pi) — the agent harness whose RPC mode and session logs make this measurable.
- The two extractors above are the whole technique; wire them behind a small CLI (`--static` vs `--session`) and you can audit an entire fleet of agents in one command.
