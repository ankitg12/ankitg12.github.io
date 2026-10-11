---
layout: post
title: "The LLM Wiki I Never Used, and the Boring Fix That Replaced It"
date: 2026-10-11
categories: ai agents productivity
series: "Moving to Linux"
---

Today I asked my agent where the shortcuts for reading Slack had gone after I moved from Windows to Linux. The answer was "nowhere — you never had any". Finding that out took the agent about fifteen tool calls. The data was all on the disk. The agent just had no map of it.

That started a clean-up of my notes folder, and a question I had asked twice before: should an LLM read all my notes and compile them into a wiki with a knowledge graph?

## The idea, and my two attempts

Andrej Karpathy described the [LLM wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f): keep raw sources unchanged, let an LLM compile them into summary pages, entity pages and cross-links, and keep a schema file that tells the LLM the conventions. Ingest, query, lint. Tools such as [OpenKB](https://github.com/VectifyAI/OpenKB) implement it.

I tried it twice:

| When | What happened |
|---|---|
| May 2026 | I designed a `~/knowledge` folder that an LLM compiler would fill from `~/notes`. I installed the tool in a venv. |
| July 2026 | An agent tested it end to end. It stopped at `init`, which asks two interactive questions and changes a global config file. The handoff note says: "scaffolded months ago but never used". |
| October 2026 | I moved to Linux. The `~/knowledge` folder did not come with me. Nobody noticed for a week. |

A tool that nobody misses has answered the question for you.

## What was actually wrong

When I measured the notes folder, the problems were not missing links between ideas:

```
root files: 166   dirs: 14
md not in index.md: 48
external hard-coded references to notes: 43, across 25 notes
```

- **Notes that described the wrong machine.** About 25 notes were only about Windows: PowerShell quoting, WSL, Scoop, Explorer menus, a Wi-Fi bug in a laptop I no longer have. They sat at the root next to the notes I use every day.
- **Wrong facts in files the agent loads every session.** The agent's instructions told it how to quote `$_` for PowerShell, and gave a `C:/Users/...` path for its own memory file. On Linux that is noise in the best case and a wrong action in the worst.
- **One file that said the laptop was "not yet returned"** and another that said it was returned. One of them was right, but the agent could not tell which until it read the dated journal entries.
- **A search tool that lied.** My agent's `grep` tool returned "No matches found" for text that was in the file. The cause: `~/Notes` is a symlink, and the tool does not follow a symlink as its search root. A short, clean, empty result is the most dangerous output a tool can give.

An LLM-compiled wiki fixes none of these. It would make the third one worse: a confident summary on top of a stale source is still stale, and now it is also fluent.

## The boring fix

1. **One locator note per retired machine.** A note named after the old laptop is a table: each archive folder on the new machine, its size, what is in it, and how to search it. One line in the agent's memory points to it. The next session that needs a Windows file reads one table instead of making fifteen calls.
2. **Move, do not delete.** Every Windows-only note went into `windows/` with `git mv`, so history is kept. A script rewrote every link to the moved files and then checked that no row in the index points to a moved file. (It also found four rows that were broken long before today.)
3. **Put a status line on dead designs.** The `~/knowledge` design stays in the notes as a record of intent, with one line on top: *not in use, never populated, not carried over*. The decision table that told agents "write durable things to `~/knowledge`" now points to places that exist.
4. **Use the old machine's scheduler as the inventory.** I did not try to remember what my Windows laptop did automatically. I compared its exported Scheduled Task XML files with `systemctl --user list-timers`. Of 11 tasks, a few already had a systemd timer, two fixed problems that only Windows had, and the rest were decisions: port it, run it by hand, or drop it. That is a list I could not have written from memory.

## The prior art is older than LLMs

Niklas Luhmann kept his notes on paper slips in his [Zettelkasten](https://niklas-luhmann-archiv.de/bestand/zettelkasten/inhaltsuebersicht). It worked because of two plain things: each slip had a fixed address, and a keyword register pointed from topics to addresses. My `index.md` is that register. When it falls behind (48 notes were missing from it), the whole system gets harder to use, however clever the layer on top is.

If I want a graph, I already have one. [Logseq](https://github.com/logseq/logseq) keeps backlinks between plain Markdown pages, and [zk](https://github.com/zk-org/zk) gives the same over a folder of notes through a language server. Neither needs a model to build it.

## What I search with instead: a tool older than the LLM era

On Windows I had set up [Recoll](https://www.recoll.org/), a full-text search tool built on Xapian. It did not come with me to Linux either. On Linux it took one `apt install` and a six-line config, with no wrapper scripts:

```
# ~/.recoll/recoll.conf
topdirs = /home/me/repos/github.com/me/notes \
          /home/me/repos/github.com/me/logseq \
          /home/me/agents
skippedNames+ = .git node_modules .venv *.db *.jsonl
```

```
$ recollindex            # about 9,800 documents in 15.7 s
$ recollq -b 'some-distinctive-token'
file:///home/me/repos/github.com/me/notes/old-laptop.md
```

Four things I would tell anyone setting it up:

1. **List real paths in `topdirs`, not symlinks.** My `~/Notes` is a symlink to a git checkout. The same symlink is what made my agent's search tool return a false "no matches".
2. **Do not write your own service.** Recoll ships a systemd user unit in `/usr/share/recoll/examples/recollindex.service`. Copy it to `~/.config/systemd/user/` and enable it. It runs `recollindex -m`, which watches the tree with inotify.
3. **Test the index three ways before you trust it.** Search for a word you know is in a file (it must be found), a nonsense word (it must not be found), and a word you just changed in a file (the new word must be found and the old one gone). An empty result proves nothing until a known word is found. In my test, a changed file was searchable within 8 seconds.
4. **Deleted files stay in the results for a while.** A deleted file still matched 25 seconds later and was gone after about 3 minutes, with no log line either way. Running plain `recollindex` while the watcher runs purges it within seconds. Until then, check that a hit still exists before you act on it.

This is the search layer the LLM wiki was supposed to give me. It needs no model, no API key and no tokens, and it reads the notes as they are, not a summary of them.

## When the LLM wiki is the right tool

The pattern still fits one job well: **external sources I am reading**, such as papers and long docs. There the raw text is not mine, a summary saves real time, and a wrong summary is easy to check against the paper. That was the scope I first wrote down for it, and it is where I will try it next.

For my own operational notes, where the exact path or flag is the whole value, the answer is an index, a folder for the past, and a status line on every design that did not happen.

## Source

- Andrej Karpathy, [LLM wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- [OpenKB](https://github.com/VectifyAI/OpenKB)
- [Niklas Luhmann-Archiv: Zettelkasten overview](https://niklas-luhmann-archiv.de/bestand/zettelkasten/inhaltsuebersicht)
- [Logseq](https://github.com/logseq/logseq), [zk](https://github.com/zk-org/zk)
- [Recoll](https://www.recoll.org/), and its manual page on [real-time indexing](https://recoll.org/usermanual/webhelp/docs/RCL.INDEXING.MONITOR.html)
