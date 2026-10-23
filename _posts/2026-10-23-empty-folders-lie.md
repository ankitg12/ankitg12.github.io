---
layout: post
title: "Empty folders lie: recursive prune with marker-aware cascade"
date: 2026-10-23
categories: tools filesystems
---

My Samsung T5 had ~90 empty folders. Finder showed nothing inside them — but `find -empty` refused to match most of them.

The lie: Android leftovers where every folder contains a 0-byte `.nomedia`, a `.database_uuid`, or a `.uid` file. Technically non-empty. Visually empty. `Movies/`, `Music/`, `Pictures/` each held exactly two marker files and nothing else.

## The two-pass trick

Delete the markers first, then cascade `rmdir` bottom-up until no pass removes anything. Parents that become empty after their children go get removed too — one pass isn't enough, because emptiness propagates upward.

```bash
# markers first (0-byte only — never touch real files)
find . -type f \( -name ".nomedia" -o -name ".database_uuid" \

# cascade until stable
while [ "$(find . -type d -empty -print -delete | wc -l)" -gt 0 ]; do :; done
```

The loop condition is the fixpoint: when a pass deletes zero dirs, the tree is clean. My first version did a single `find -delete` and left the whole `Android/data/com.*` shell tree standing.

## Rules that survived contact with a real drive

- **Dry-run default.** The script prints `rmdir:` lines; `--apply` deletes. I previewed on the T5 before touching anything.
- **Symlink = content.** Never follow; a symlinked dir keeps its parent alive. `os.walk(followlinks=False)` + `os.scandir` checks.
- **`os.rmdir`, never `shutil.rmtree`.** `rmdir` fails on non-empty — safe against races and bugs.
- **Skip system dirs** (`.Spotlight-V100`, `.fseventsd`, `System Volume Information`) and **refuse protected roots** (`/`, `C:\`, `$HOME`).
- **OS junk is per-platform:** `.DS_Store` (macOS), `Thumbs.db`/`desktop.ini` (Windows), `.nomedia` (Android).

## The script

Python, stdlib only, macOS/Windows/Linux: [prune-empty-dirs.py gist](https://gist.github.com/ankitg12/604342887516c2643b4bb2511c8fe94e). A bash port for Mac/Linux follows the same logic. Same flags:

```bash
prune-empty-dirs.py --prune-markers /Volumes/Samsung_T5   # preview
prune-empty-dirs.py --apply --prune-markers ~/old-phone-dump
```

Verified on a fixture: 9 dirs + 2 markers removed, `keep-real/` and symlink parents untouched, recount confirms 0 empties remain. Final dry-run on the T5: `0 dir(s) + 0 marker(s)` — drive top level went from 14 entries to 4.

## When not to build

Pure-empty cleanup already has tools: `find -empty -delete` (built-in), [RED](https://www.jonasjohn.de/red.htm) (Windows), [Czkawka](https://github.com/qarmin/czkawka) (cross-platform), [SD Maid SE](https://play.google.com/store/apps/details?id=eu.darken.sdmse&hl=en_US) / [Files by Google](https://support.google.com/files/answer/9848742?hl=en) / [Empty Folder Cleaner](https://play.google.com/store/apps/details?hl=en_US&id=com.emptyfolder.emptyfolderremover.emptyfoldercleaner) (Android on-device). Build only for the marker-aware cascade — that's the gap.
