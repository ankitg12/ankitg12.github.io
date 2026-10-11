---
layout: post
title: "Two Notebooks on Purpose: Why My Notes Live in Logseq and in a Plain Folder"
date: 2026-10-11
categories: tools productivity logseq
series: "Personal Productivity"
---

I installed [zk](https://github.com/zk-org/zk) today and had to tell it which notebook to use. I have two: a plain folder of Markdown notes that I started first, and a [Logseq](https://github.com/logseq/logseq) graph that I picked up later. The obvious question followed: should I merge them, and if so, which way?

I decided not to merge them. This post explains why, and shows the small amount of work that made two notebooks a design choice instead of an accident.

It continues the clean-up in {% post_url 2026-10-11-the-llm-wiki-i-never-used %}.

## The two trees, measured

I did not want to argue from impressions, so I counted first:

| | Notes folder | Logseq graph |
|---|---|---|
| Markdown files in git | 416 | 221 |
| Size on disk (without `.git`) | 5.1 MB | 14 MB |
| Commits in the last 30 days | 152 | 128 to `journals/`, 17 to `pages/` |
| Files that link to other notes | 19 | 63 (all `[[wiki-links]]`) |
| Format | Prose, YAML frontmatter, plus `.db`, `.json`, `.pdf`, `.typ` files | Outliner bullets (`- `) |
| Main reader | My coding agents, by file path | Me, through the Logseq app |
| Indexed by | **Topic**: how a thing works | **Time**: what happened on a day |

The last row is the important one. The two trees are not two copies of one notebook. One is a journal and the other is a reference manual. A lab notebook and a design spec are kept apart for the same reason.

The second-to-last row matters almost as much. About 40 files outside the notes folder (agent instructions, skills, memory files, scripts) point to notes by path, for example `~/Notes/omp/install-config.md`. Logseq finds a page by its title and its `[[links]]`. Those are two different ways to address a note, and each suits its reader.

## Why not move everything into Logseq?

My first guess was "Logseq cannot scale to reference notes". That guess was wrong. My whole notes folder is about 5 MB. The Logseq problems I found reported online start with graphs hundreds of times larger. Size was never the issue.

The real reasons come from what kind of system Logseq is becoming. The Logseq team's own notes on the [DB version](https://github.com/logseq/docs/blob/master/db-version-changes.md) say:

```
File graphs have been split off into https://github.com/logseq/og ...
The application performance is better - loading faster, handling larger graphs ...
Markdown syntax for blocks e.g. a heading or quote is no longer visible or editable.
db.sqlite - Stores all your graph's data including user configs.
```

That gives four reasons:

1. **The file version is no longer the main product.** The Markdown-file Logseq that I use now lives in a separate repository, [logseq/og](https://github.com/logseq/og) ("Logseq og (file version)"). The main line stores everything in SQLite. If my agents' reference notes lived inside Logseq, their source of truth would depend on a branch the makers have set aside.
2. **The app owns the format.** Logseq is an outliner. Every line is a block, and the app writes files back in its own shape. A note that agents read by path needs a format that does not change under them.
3. **The page model is flat.** Logseq turns folders into `a___b` filenames, and attachments go to `assets/`. My reference notes use real folders (`linux/`, `windows/`, `omp/`) and keep non-Markdown files next to the notes that explain them.
4. **The tools I already use need plain text.** `grep`, `ripgrep`, `zk`, `git diff`, a full-text indexer, and every coding agent read the folder directly. They cannot read a SQLite file in the same way.

## Why not move Logseq into the notes folder?

That way round loses the journal app, which is the reason I picked up Logseq. My daily task CLI, my journal-append command, and an hourly job that writes a digest page per day all assume Logseq's layout (`journals/YYYY_MM_DD.md`, `pages/`). Rewriting them would have a real cost and give nothing back.

## What I did instead: write down the boundary

Two homes are fine if every note has exactly one of them. So the work was to write the rule down and then find out where reality broke it.

The rule now sits in the notes folder's own conventions file:

```markdown
### Logseq vs ~/Notes
- Logseq is indexed by time: journals, daily tasks, generated daily
  digest pages, and personal threads.
- ~/Notes is indexed by topic: reusable technical reference.
  Register every file in index.md.
- The bridge: a journal entry links to a ~/Notes path; it never copies
  the content. A topic page found in Logseq pages/ belongs in ~/Notes.
```

Then I audited the Logseq `pages/` folder, leaving out the generated daily digests:

```
$ ls pages | grep -v '^daylog___'
Tasks.md
Linux Multi-Queue Networking & Hardware Queues.md
<personal-thread>___...    (2 pages)
<career-plan>.md
omp.md
agent-skills-setup.md
contents.md
```

That is only eight pages. Logseq's `pages/` folder had not been used as a reference store for weeks, so the split was already happening without anyone deciding it. Four pages broke the rule:

| Page | What it was | Action |
|---|---|---|
| A tool install note | Notes from my old Windows laptop | Moved to `~/Notes/windows/` |
| A skills-sharing note | Windows junction setup, same era | Moved to `~/Notes/windows/` |
| Linux multi-queue networking | Technical reference | Moved to `~/Notes/linux/` |
| A career-planning note | **One of three copies** of the same plan | Merged into the one canonical file; the other two became one-line pointers |

Two details made the moves safe:

- **Check that the move did not lose content.** Each moved file got one provenance line at the top ("Moved from Logseq `pages/…` on 2026-10-11"). After that line, its bytes must match the original as it was in git:

  ```bash
  git show "HEAD:pages/$src" | cmp - <(tail -n +3 "$NOTES/$dst") && echo IDENTICAL
  ```

- **Leave a pointer, not a gap.** The old location of the merged career note is now a single bullet that names the canonical file. The old text is still in git history.

The duplicate was the real find. One plan in three files means three versions that drift apart, and an agent that reads one cannot tell whether it is the current one. In a single store that is a smaller risk, but it is the same failure.

## A correction worth keeping

My first audit said two of those pages had no counterpart in the notes folder. That was wrong for one of them: a third copy of the career note existed. My agent's search tool does not follow a symlink as its search root, and `~/Notes` is a symlink to a git checkout. I made the same mistake in {% post_url 2026-10-11-the-llm-wiki-i-never-used %}, on the same day. The fix is the same as before: search the real path, and treat a short, empty result as unproven until a known word is found.

## zk on top

zk works on either tree, because both are plain Markdown. I made Logseq the default notebook, because that is where the `[[links]]` are. Only 19 of 416 files in the notes folder link to other notes, so a link graph adds little there today.

```bash
# ~/.bashrc
export ZK_NOTEBOOK_DIR="$HOME/Logseq"
```

```
$ zk index
  + 227 added
$ cd /tmp && zk list --limit 2
Found 2 notes
```

## The older idea underneath

In 1978 Doug McIlroy wrote down the rules of the Unix style. In 1994 Peter Salus summarized them, crediting McIlroy, as three lines, and the third is:

> Write programs to handle text streams, because that is a universal interface.

That is the real argument for two notebooks. My journal can live in an app that has opinions about storage, because I am its main reader. My reference notes are read by many programs I have not written yet, so they stay as plain text in folders. Logseq's move to SQLite is a sensible trade for an app. For a store that agents and tools read, plain text is the right side of that trade.

## Source

- [zk](https://github.com/zk-org/zk), a plain-text notebook CLI (project since 2020)
- [Logseq](https://github.com/logseq/logseq), and [logseq/og](https://github.com/logseq/og), the file-based version
- Logseq docs: [DB version changes](https://github.com/logseq/docs/blob/master/db-version-changes.md)
- Wikipedia: [Unix philosophy](https://en.wikipedia.org/wiki/Unix_philosophy) (McIlroy 1978, summarized by Salus 1994)
- Wikipedia: [Zettelkasten](https://en.wikipedia.org/wiki/Zettelkasten)
- Previous post: {% post_url 2026-10-11-the-llm-wiki-i-never-used %}
