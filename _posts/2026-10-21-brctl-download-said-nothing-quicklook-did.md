---
layout: post
title: "`brctl download` Said Nothing. QuickLook Downloaded the File."
date: 2026-10-21
categories: macos icloud debugging
---

`cp` on a 4 MB screenshot failed with `Resource deadlock avoided`. Finder showed it. `ls` showed its size. `mdls` knew it was a PNG. But nothing could read a byte of it — and the documented fix didn't work either.

## The symptom

Five screenshots on an iCloud-synced Desktop. Four evicted to the cloud (dataless placeholders), one local. Every read path failed identically:

```text
$ cp "Screenshot 2026-10-20 at 4.51.38 PM.png" /tmp/s2.png
cp: error reading 'Screenshot 2026-10-20 at 4.51.38 PM.png': Resource deadlock avoided

$ dd if="Screenshot 2026-06-30 at 4.32.29 PM.png" of=/tmp/s1.png bs=64k
dd: error reading 'Screenshot 2026-06-30 at 4.32.29 PM.png': Resource deadlock avoided

$ sips -g pixelWidth "Screenshot 2026-10-20 at 5.03.52 PM.png"
  pixelWidth: <nil>
```

`EDEADLK` on a `cp`. The metadata layer (`mdls`, `ls -l`) reports everything correctly — size, creation date, PNG type — because metadata syncs separately from file contents. Only byte reads hit the deadlock.

## The wrong fix I tried first

The published recipe for this (rsync-vs-iCloud writeups) is `brctl download` followed by a small read to confirm materialization:

```text
$ brctl download "Screenshot 2026-10-20 at 4.51.38 PM.png"
$ head -c 1 "Screenshot 2026-10-20 at 4.51.38 PM.png" >/dev/null && echo OK
FAIL
```

Still `dataless`. I then ran `brctl evict .` on the whole directory by mistake — wrong direction, evicted the one local file too. `brctl download ~/Desktop` afterward didn't move anything either; `sips` still returned nil. The daemon had the request and sat on it.

## What actually worked

`qlmanage -t` — the QuickLook thumbnail generator — materialized every file it touched:

```text
$ qlmanage -t -s 800 -o /tmp "Screenshot 2026-10-20 at 4.51.38 PM.png" >/dev/null 2>&1
$ head -c 1 "Screenshot 2026-10-20 at 4.51.38 PM.png" >/dev/null && echo READ-OK
READ-OK
```

The `dataless` flag cleared on `ls -lO` immediately after. One file per `qlmanage` call, then `head -c 1` to confirm. Three screenshots, three rounds, all readable.

Why this works when `brctl download` didn't: QuickLook goes through the file-coordination path (the same one Finder uses when you click an evicted file), which blocks until the bytes arrive. A bare `brctl download` from a shell queues the fetch but nothing waits on it — and a synchronous `head` on the still-dataless file deadlocks instead of waiting. The thumbnail render *is* the wait.

## The detection one-liner

`find -flags +dataless` is the published detector. It works, but BSD `ls` already shows it — and `stat`/`ls -O` from muscle memory will bite:

| Command | Result on macOS |
|---|---|
| `/bin/ls -lO` (capital O) | Shows `dataless` flag — this is the one |
| `ls -lO` via GNU coreutils (brew) | `error: unexpected argument '-O'` — GNU `ls` has no `-O` |
| `stat -f '%f %N'` (BSD flags) | `unsupported BSD format directive` under coreutils `stat` |

If you installed coreutils, your `ls` and `stat` are GNU and the BSD-only flags vanish. Use `/bin/ls -lO` with the full path, or check whether you're shadowed first (`which ls`).

## Gotchas

- **`brctl evict` is the opposite of what you want.** It evicts local copies. On a whole directory it takes the files you *can* read offline with it. There is no undo flag — only re-download.
- **`brctl download` queues; it doesn't wait.** Follow it with a blocking read through a coordinated path (QuickLook, Finder click, `open`), not a raw `head`.
- **Metadata lies by design.** `ls` size, `mdls` type, `sips` nil — the first two come from the sync database, which is always local. Only a byte read proves the contents are on disk.
- **Filenames with narrow no-break spaces.** macOS screenshots use U+202F (`Screenshot 2026-10-20 at 5.03.52 PM.png`), not a regular space before PM. Tab-complete them; don't retype.
- **Desktop hiding is unrelated to any of this.** `defaults write com.apple.finder CreateDesktop -bool false` hides icons; the files and their eviction state are untouched.

## Source

- Prior art on the rsync side: [rsync kept dying on iCloud's evicted files](https://blackboxblast.com/en/icloud-dataless-rsync-deadlock/) — `find -flags +dataless` + `brctl download` + `head -c 1`. Correct detection, but the download step alone didn't materialize here.
- The agent-hang angle: [The silent enemy of local agents: iCloud Deadlocks](https://mariohealthbits.dev/en/blog/2026-08-05-icloud-deadlocks-agentes/) — `brctl download` before synchronous reads.
- Unanswered upstream (bot-walled against curl, loads in browsers and readers): [macOS launchd script vs evicted file](https://stackoverflow.com/questions/79321973/macos-trying-to-load-evicted-file-from-launchd-shell-script-leading-to-resour) — same EDEADLK, no accepted answer.
- `man brctl` documents `download`/`evict`/`dump`/`monitor`; it says nothing about blocking semantics, which is the entire problem.
