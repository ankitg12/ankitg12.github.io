---
layout: post
title: "Search Should Widen, Not Explode"
date: 2026-07-30
categories: ai agents productivity tools
series: "AI coding agent productivity"
---

`es todo.md` returned the file I wanted—and hundreds of copies from package caches, dependency trees, backups, and generated workspaces.

The search was fast. It was also poorly posed.

This distinction matters more for coding agents than for humans. A human can scan a noisy result window, notice that most entries are irrelevant, and refine the query. An agent may ingest the entire result, spend context on duplicate paths, or compensate with more shell pipelines and retries. A tool that answers in milliseconds can still waste minutes of reasoning.

The fix was not a faster index. It was **progressive discovery**: search the highest-signal locations first, bound the output, and make widening explicit.

## Everything was doing exactly what I asked

[Everything](https://www.voidtools.com/) is exceptionally good at filename and path lookup on Windows. On NTFS volumes, it builds its index from filesystem metadata and tracks changes efficiently. Renames and newly created files appear almost immediately.

That makes a command such as this feel harmless:

```text
es todo.md
```

But the command means:

> Search the complete indexed filesystem for every filename containing `todo.md`.

It does not mean:

> Find the human-authored TODO document most relevant to my current task.

Everything has no reason to prefer a document under `Documents` over an identically named file under a package cache. It also does not read `.gitignore`; Git's ignore convention is not a filesystem-search convention.

The core mistake is confusing **retrieval speed** with **result relevance**.

## Evidence from real agent usage

I queried local coding-agent tool-call history rather than judging the problem from one bad search. Across 301 sessions, agents had made 1,317 calls to `es`.

|Pattern|Observed|
|---|---:|
|Calls without a result limit such as `-n`|1,267 (96.2%)|
|Calls using the explicit `-path` option|202 (15.3%)|
|Sessions containing multiple `es` calls|225 of 301 (74.8%)|
|Maximum `es` calls in one session|21|
|Largest captured result|about 51 KB|

There were several recurring failure modes:

1. **Unbounded global queries** for short terms.
2. **Filtering after retrieval**, often by piping a large result through another command.
3. **Retrying query variants** after an empty or noisy result.
4. **Literal environment variables** passed into Everything even when environment expansion was disabled.
5. **Ambiguous `path:` expressions** whose quoting and argument boundaries were easy to get wrong.
6. **One quoted multi-token expression**, such as `"F16 ext:pdf"`, where the CLI expected separate search arguments.

That last mistake is subtle:

```text
# Wrong: interpreted as one phrase
es "F16 ext:pdf"

# Right: filename term plus an Everything search function
es F16 ext:pdf
```

The audit changed the design target. This was not one user's query-hygiene problem; it was a repeated interface failure across agents.

## The 80/20 fix

Most of the value comes from two rules:

1. **Scope before searching.**
2. **Bound before printing.**

If the likely root is known, raw `es` is already sufficient:

```text
es -n 50 -path "C:\Users\<username>\Documents" /a-d F16 ext:pdf
```

This says:

- return at most 50 entries;
- search recursively below a known root;
- return files, not directories;
- require `F16` in the name;
- require a PDF extension.

The query is both faster to consume and more honest about its uncertainty.

The remaining 20% is what to do when the root is *not* known. That is where progressive discovery helps.

## A widening ladder

I built a thin command called `esq` around the established Everything engine. It does not implement indexing or search. It only supplies a retrieval policy shared by humans and agents.

```text
esq QUERY
```

The policy has four explicit levels:

|Level|Scope|Purpose|
|---:|---|---|
|1|Personal documents, downloads, notes, and cloud-synced folders|Highest probability of human-authored artifacts|
|2|Level 1 plus project, repository, agent, and tool roots|Development work without dependency noise|
|3|Complete index with common caches and generated trees omitted|Broad recovery while retaining useful filesystem coverage|
|4|Raw Everything index|Forensics and last-resort discovery|

A level-1 search ends with a widening hint:

```text
Level 1: personal files — 8 result(s)
...

Widen: esq --level 2 todo.md
```

The next search is a conscious decision rather than an automatic explosion:

```text
esq --level 2 todo.md
esq --level 3 todo.md
esq --level 4 todo.md
```

This is progressive disclosure applied to retrieval: expose complexity only when the previous layer is insufficient.


## Make the safe operation easier

A guardrail that depends on remembering five flags will eventually fail. The safer operation must also be the easier one.

`esq` therefore provides structured options instead of expecting callers to compose fragile Everything expressions:

```text
# A bounded search under a path; environment variables are expanded first
esq --path "%USERPROFILE%\Documents" -n 20 F16

# Folders instead of files
esq --folders project-name

# Both files and directories
esq --both architecture

# Stable output for an agent
esq --json -n 10 todo.md
```

It rejects ambiguous inline `path:` expressions and points the caller to `--path` instead. The wrapper also derives home-relative roots at runtime, so it does not embed a particular username.

This is an important agent-tool design principle:

> Turn a convention into an interface whenever violating the convention is cheap and common.

## Why not just configure exclusions?

Everything provides useful native mechanisms, particularly [Result Omissions in Everything 1.5](https://www.voidtools.com/support/everything/result_omissions/). Omissions can hide system and cache results without removing them from the underlying index.

They are valuable, but they solve only part of the problem:

- A global omission list does not know the current task's likely root.
- Permanent index exclusions can hide files needed during diagnosis.
- GUI filters are convenient for humans but are not consistently exposed through the ES command-line client.
- Agents still need bounded, machine-readable output and an explicit widening protocol.

The best arrangement is layered:

1. Everything indexes broadly.
2. Native omissions suppress universal noise.
3. `esq` selects a task-appropriate tier and bounds output.
4. Raw `es` remains available when the path or exact filename is already known.

## Why not use another search interface?

I checked established alternatives before retaining custom code.

|Tool|Strength|Why it did not replace the widening layer|
|---|---|---|
|[fzf](https://github.com/junegunn/fzf)|Mature interactive fuzzy selection|Selects from supplied candidates; it does not define filesystem scope or agent output policy|
|[EverythingToolbar](https://github.com/srwi/EverythingToolbar)|Excellent human GUI integration|Not a deterministic agent interface|
|[es-tui](https://github.com/Foadsf/es-tui)|Interactive terminal UI over Everything|No staged omission policy or machine-readable agent mode; much smaller adoption history|
|Windows Search|Indexes document content and properties|Different retrieval problem; lacks an equally simple whole-filesystem CLI|

The custom part remained small because it orchestrates a mature search engine instead of replacing one.

## Everything versus Windows Search

Everything is not universally better than native Windows Search.

|Question|Prefer|
|---|---|
|“Where is the file whose name contains `Form16`?”|Everything|
|“Which workbook contains this account number?”|Windows Search content index|
|“What was renamed five seconds ago?”|Everything|
|“Which PDF mentions a particular clause?”|Windows Search, provided a suitable PDF filter indexed it|
|“Show every matching path on all indexed NTFS volumes.”|Everything|
|“Search Office properties, authors, and document contents.”|Windows Search|

Everything is primarily a **filename/path index**. Windows Search is primarily a **content/property index**. A good retrieval workflow uses both rather than declaring one the winner.

## Similar patterns elsewhere

Progressive file search is an instance of a broader systems pattern: delay expensive fan-out until evidence justifies it.

- **Database query planning:** use selective predicates before wide joins.
- **Network congestion control:** increase the window after successful delivery rather than transmitting at maximum rate immediately.
- **Observability:** begin with aggregate signals, then drill into traces and raw events.
- **Incident response:** inspect the most likely subsystem before collecting every log on every host.
- **LLM retrieval:** retrieve a small relevant set, then expand only when confidence is low.

There is also a connection to [Maximum Marginal Relevance](https://www.elastic.co/search-labs/blog/maximum-marginal-relevance-diversify-results): ten near-identical dependency copies provide less information than a smaller set drawn from distinct directories. A future search layer could diversify results by parent directory, not merely deduplicate exact paths.

## Design rules for agent-facing search

The exercise produced a compact checklist:

1. Default to the smallest plausible scope.
2. Put a hard ceiling on displayed results.
3. Filter at the source, not after retrieval.
4. Make widening explicit and monotonic.
5. Preserve an escape hatch to raw data.
6. Expose the same mental model to humans and agents.
7. Provide structured output for automation.
8. Expand portable path variables before invoking tools that may not.
9. Reject ambiguous syntax with a corrective error.
10. Audit real tool history; interface defects hide in repeated workarounds.

The non-technical version is one sentence:

> A good search starts in the most likely drawer and opens the rest of the house only when necessary.

The hardest part is explaining why an instant answer can still be expensive. The machine spends almost no time finding the files; the human or agent pays to interpret everything it found.

## Update — one query, a complete index

The first `esq` implementation searched each level-1 root separately. Eight roots meant eight sequential `es.exe` processes, and a representative search took about 14 seconds. The wrapper now sends one OR-grouped path query instead:

```text
<path:"Documents"|path:"Downloads"|path:"Notes"> regcool
```

The real paths are absolute and derived at runtime; the abbreviated example shows only the query structure. The same three expected results now return in roughly 4–5 seconds. A raw `es` query took about 2.3 seconds, so replacing this small Python wrapper with another language would not attack the dominant cost. Direct SDK/IPC integration remains possible, but the measured latency does not yet justify that complexity.

The invocation also uses the official ES client's `-argv` mode, which applies Windows `CommandLineToArgvW` parsing so paths containing spaces remain correctly separated arguments.

The other correction was at the indexing layer. Excluding `Program Files`, `AppData`, and `Windows` globally kept results quiet, but it made `esq --level 4` incapable of finding files that never entered Everything's index. Those exclusions were removed. The base index is now complete; levels 1–3 own relevance and noise control at query time, while level 4 remains a genuine raw-index escape hatch.

The refined rule is: **indexing answers what exists; the query layer decides what is useful now.**

## Source

- [Everything](https://www.voidtools.com/)
- [Official Everything command-line client source](https://github.com/voidtools/ES)
- [Everything documentation](https://www.voidtools.com/support/everything/)
- [Everything 1.5 Result Omissions](https://www.voidtools.com/support/everything/result_omissions/)
- [Windows Search documentation](https://learn.microsoft.com/en-us/windows/win32/search/windows-search)
- [fzf](https://github.com/junegunn/fzf)
- [EverythingToolbar](https://github.com/srwi/EverythingToolbar)
- [es-tui](https://github.com/Foadsf/es-tui)
- [Diversifying search results with Maximum Marginal Relevance](https://www.elastic.co/search-labs/blog/maximum-marginal-relevance-diversify-results)
