---
layout: post
title: "A memory feature that's capped by count, not by usefulness"
date: 2026-07-08
categories: ai agents productivity
series: "AI coding agent productivity"
---

A small conversation — a handful of turns, no large files read — was already at 7.3% of a 1M-token context window. That's over 70,000 tokens before the actual work started. Something was being injected into every session that shouldn't have been growing unbounded.

This post is about finding it, and a subtler bug than "unbounded growth": a memory feature that *is* capped — but capped by raw entry count, with no compression or dedup logic ever running on what it retains. The cap doesn't do the job a cap is supposed to do.

## The setup: two memory systems, one gets consolidated

OMP (`oh-my-pi`, a Claude Code-style coding agent) ships an autonomous memory backend. Enable it and it does two independent things per project:

**1. A pipeline with real consolidation.** A background job reads past sessions, extracts durable facts (Phase 1), then a second pass merges everything into a curated long-term doc and a short injected summary (Phase 2):

```
raw_memories.md  →  MEMORY.md  (curated, full)
                 →  memory_summary.md  (compact, injected at session start)
```

The injected summary has a hard cap — `memories.summaryInjectionTokenLimit`, default 5000 tokens. No matter how much raw material accumulates, what actually lands in your context stays bounded. This is the correct design.

**2. A `learn` tool with a count cap and nothing else.** Separately, the agent (or you) can call a `learn` tool mid-session to record a fact worth keeping — "this API needs header X," "this repo's pre-commit hook does Y." These append to a flat file, `learned.md`, and the *entire file* gets injected into every future session's system prompt as a "Learned lessons" block.

Reading the actual source (not just the docs) turned up the real design:

```ts
const MAX_LEARNED_LESSONS = 100;            // newest-first cap on entry count
const MAX_LEARNED_CONTENT_CHARS = 2000;     // per-entry char cap
const MAX_LEARNED_CONTEXT_CHARS = 400;      // per-entry context-field cap

// on append: filter exact-string duplicates, prepend, then
// slice(0, MAX_LEARNED_LESSONS) — oldest entries fall off the end
```

So it isn't uncapped. It's capped at 100 entries, each up to 2,400 bytes, with an *exact-string-match* dedup pass on write. My first draft of this post claimed "no cap, no consolidation, no dedup, just append forever" — inferred from a doc that only described the *other* memory path's cap, not verified against this code path. That was wrong, and worth naming: measuring file sizes told me *something* was broken, but only reading the actual eviction logic told me *what*.

## Finding the actual number, not guessing

The instinct when someone says "context is bloated" is to eyeball the obvious suspects — long project instruction files, a big skills list, verbose tool descriptions. All plausible, all wrong here. Measure first:

```python
import os
for name in ["learned.md", "raw_memories.md", "MEMORY.md", "memory_summary.md"]:
    p = f"~/.omp/agent/memories/--<project-slug>--/{name}"
    n = os.path.getsize(os.path.expanduser(p))
    print(f"{name}: {n} bytes (~{n//4} tokens)")
```

Results for one active project, after several months of use:

| File | Size | Est. tokens |
|---|---|---|
| `memory_summary.md` (capped, pipeline output) | 450 bytes | ~110 |
| `MEMORY.md` (curated, pipeline output) | 12 KB | ~3,000 |
| `raw_memories.md` (pipeline input) | 108 KB | ~27,000 |
| **`learned.md` (uncapped, `learn` tool output)** | **133 KB** | **~33,000** |

`learned.md` alone was the single largest thing in context — bigger than the entire consolidated memory pipeline combined, and bigger than the project's own instruction file (~1,600 tokens). A feature meant to save a few hundred tokens of re-discovery per fact had, at 100 accumulated entries, become the dominant cost of every session.

The natural instinct — "run the rebuild command" — doesn't touch it. `/memory rebuild` reprocesses `raw_memories.md` through the Phase 1/2 pipeline. `learned.md` is a completely separate code path with its own template variable (`{{learned}}` vs `{{memory_summary}}` in the prompt template) and no rebuild hook of its own.

And the entry-count cap doesn't help here, because eviction is pure FIFO-by-age: it drops the *oldest* entry once you're past 100, regardless of whether that entry is stale, redundant, or the single most useful thing in the file. A verbose, narrative, half-duplicate entry written last week outranks a terse, load-bearing fact written six months ago — purely because it's newer. The cap bounds *disk size*; it does nothing for *signal density*, which is the thing that actually costs context tokens.

## Why it grew this way: content shape, not just count

100 entries isn't inherently a lot. The problem was entry *shape*. A quick length histogram:

```python
entries = split_on_leading_dash_bullets(text)  # 100 entries
long = [e for e in entries if len(e) > 900]
print(len(long), "entries over 900 chars")   # → 68
print(sum(len(e) for e in long))              # → 112,658 of 131,997 total chars
```

68 of 100 entries — 85% of the file's bytes — were long-form narrative: "On 2026-07-06, root-caused X after trying Y, which didn't work, then discovered Z via method W, confirmed via..." That's a session diary entry, not a durable fact. A `learn` call captures whatever the agent decided was worth remembering in the moment, and an agent mid-investigation tends to narrate its process, not distill the takeaway.

There was also near-duplication the write-time dedup couldn't catch — two entries recording the same environment-variable fix, one a strict superset of the other, worded slightly differently. Exact-string matching on write only catches a lesson re-saved verbatim; it can't catch "the same fact, restated."

## The fix: batch-compress, don't hand-edit

Rewriting 68 verbose entries by hand isn't worth anyone's time. This is a good fit for exactly what these agents are for — a bulk semantic-preserving rewrite:

```python
def compress_batch(entries_subset):
    numbered = "\n\n".join(f"[{i}]\n{e}" for i, e in entries_subset)
    prompt = f"""Rewrite EACH entry below as a terse, durable, self-contained
factual bullet (2-4 sentences). Keep concrete facts — paths, commands, config
keys, root causes, numbers. Strip session narrative and investigation steps.
If an entry says it supersedes/corrects an earlier one, state the corrected
fact plainly. If an entry is genuinely stale, output DROP instead.

Output format, one per entry:
[N] <compressed bullet>
or
[N] DROP

Entries:
{numbered}"""
    return completion(prompt, model="default")

batches = chunk(long_entries, size=8)
results = parallel([lambda b=b: compress_batch(b) for b in batches])
```

Batching in groups of 8 kept each call well within a normal context window and let them run concurrently. Parsing back was a simple `[N] ...` tag match. Result:

```
100 entries → 92 entries  (1 exact duplicate, 7 explicitly stale, dropped)
132 KB / ~33,000 tokens → 65 KB / ~16,200 tokens
```

A 51% reduction with no information loss — spot-checked several compressed entries against the originals; every concrete fact (file path, command, config key, root cause) survived, only the narrative scaffolding was cut.

```python
new_text = "\n\n".join(kept_entries) + "\n"
with open(learned_md_path, "w", encoding="utf-8") as f:
    f.write(new_text)
```

`learned.md` is a plain file on disk (the `memory://` internal URL scheme is read-only, but the underlying path isn't) — safe to edit directly once you've verified the rewrite preserved structure.

## The general lesson

"It has a cap" and "it's bounded in a way that matters" are different claims, and it's worth separating them for any tool with an "agent remembers things" feature. A count-based FIFO cap answers "will this consume unbounded disk/context eventually" (no) but not "will the oldest, most load-bearing fact survive as long as the newest, most verbose one" (also no — age is the only eviction signal). If a memory mechanism's cap is pure count-or-size with no relevance/quality signal in the eviction decision, expect the same failure mode this file had: fine for a while, then dominated by whichever recent entries happened to be written verbosely, with genuinely durable older facts one `learn` call away from silently falling off the end. Worth a periodic manual compaction pass regardless of whether the cap exists — the cap prevents unbounded growth, not quality decay.

## Source

- OMP (`oh-my-pi`): [github.com/can1357/oh-my-pi](https://github.com/can1357/oh-my-pi)
- Memory pipeline docs: `omp://memory.md` (internal doc URL, ships with the tool)
</content>
<parameter name="i">Write blog post on OMP memory bloat finding