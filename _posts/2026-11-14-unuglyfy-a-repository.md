---
layout: post
title: "Unuglyfy a Repository: From a Crowded Notes Folder to a UUG Skill"
date: 2026-11-14
categories: ai agents productivity
---

My notes repository had become difficult to navigate: reference files had accumulated in the root, the tool manifest had grown, and the index had to point across an increasingly mixed collection of subjects. The information existed, but finding the right home for it took work. A cleanup could improve it once; the harder question was how to keep it navigable as new notes arrived and apply the same discipline elsewhere.

I started by grouping notes by subject, splitting the tool manifest, and updating the index. I removed obsolete redirect files instead of leaving an old path that merely said “this moved.” Then I added a link check and a size guard. After that cleanup, I asked whether the same ideas applied to another repository and whether a reusable agent skill could carry the method. That became **UUG**, short for “unuglyfy.”

## The problem is not just file size

A large file has a mechanical cost: finding a subject requires scanning more material. A large index can also become a second copy of the knowledge instead of a map to it. The [Diátaxis documentation framework](https://diataxis.fr/start-here/) separates kinds of documentation by what a reader needs; the useful parallel here is to separate navigation from detailed subject matter, not to put every topic into the same page.

Duplication creates a different problem. If the same fact appears in two places, an edit to one copy does not update the other. My rule is **one home per fact**: an index can describe and link to the source, but should not become another authority for its contents.

Historical material needs judgment, not a blanket rule. Evidence or history with current value or a retention requirement stays. Duplicate and obsolete material can go. A redirect-only file stays only if an identified consumer needs that compatibility path. Before moving a file, search for links, fragments, code references, and external consumers.

## What changed in the notes repository

| Problem | Change | Boundary |
|---|---|---|
| Mixed root and a growing manifest | Grouped notes by subject and split the manifest into a short router and topic files | An index points to detail; it does not duplicate it |
| Old redirect files | Removed obsolete redirects and updated references | Keep compatibility paths when a known consumer needs them |
| Links after moves | Checked tracked Markdown with [Lychee](https://lychee.cli.rs/) | Offline checks do not validate remote pages or paths in code spans |
| Pages growing again | Added a staged-file size guard to the existing hook | History and evidence directories are outside the guard |

For local Markdown, Lychee can check both file paths and [fragment identifiers](https://lychee.cli.rs/recipes/anchors/). The notes repository uses:

```sh
lychee --offline --no-progress --include-fragments --root-dir / $(git ls-files '*.md')
```

A successful offline check is not a complete link audit. [Root-relative URLs](https://lychee.cli.rs/recipes/root-dir/) depend on the configured root, remote URLs are outside offline mode, and code-span paths are not Markdown links. Those need separate review.

## Why an agent skill

The cleanup was not just a list of files to move. It involved looking at the worktree, finding consumers, deciding whether a compatibility path mattered, choosing what history to retain, and checking the result. When I wanted to apply it to another repository, copying the exact folder layout or limits would have been the wrong abstraction.

The UUG skill captures the **decision process**, not a fixed layout: inspect before changing, keep one home per fact, update consumers, check links, and use judgment about removals. It also describes where a deterministic guard helps. Repository-specific limits remain repository-specific.

That distinction matters. A hook can count lines and bytes; it cannot decide whether an old piece of evidence is useful. The agent and its human make that decision.

## The staged-file trap

A size guard must check what will be committed, not merely what is on disk. In [Git's interactive staging](https://git-scm.com/book/en/v2/Git-Tools-Interactive-Staging), a file can have staged changes and additional unstaged changes. Checking only the worktree can reject a valid staged file or accept an oversized one.

The guard lists staged paths and reads each staged blob from the index. Git documents the [`:<path>` revision syntax](https://git-scm.com/docs/gitrevisions):

```sh
git diff --cached --name-only --diff-filter=ACMR --no-renames -z
git show :README.md
```

These commands show the two parts of the mechanism: discover staged paths, then read the staged content. The implementation must handle filenames safely; the `-z` form uses NUL delimiters. Deletions are excluded from the size check because there is no staged file to measure.

A second repository already had a [pre-commit hook](https://git-scm.com/docs/githooks). I extended it rather than replacing its existing check. Its guard checks four navigation pages, each against 100 lines and 12 KiB; it does not impose that limit on every document. The tests cover oversized files, unrelated files, deletion, and both directions of the staged-versus-worktree difference. One test runs an actual commit with the hook installed in a temporary repository.

That integration test exposed an environment detail: Git ignored the test hook when the temporary directory was on a `noexec` filesystem. Running the test repository on an executable filesystem allowed the hook to run. A test that only calls the Python checker would not have caught that hook-execution problem.

## What UUG enforces—and what it does not

UUG recommends inspecting existing tools and hooks before adding another. A justified size guard should name its files and limits, report the measured size on failure, check staged blobs, and have tests for both the checker and hook invocation. It should preserve unrelated work and existing hook checks.

It does **not** demand a redirect for every move, preserve all historical text, or apply one size limit to every repository. Those choices require context. The reusable part is the method for making and checking them.

## Source

- [Diátaxis: start here](https://diataxis.fr/start-here/)
- [Git hooks](https://git-scm.com/docs/githooks)
- [Git revisions and index syntax](https://git-scm.com/docs/gitrevisions)
- [Git interactive staging](https://git-scm.com/book/en/v2/Git-Tools-Interactive-Staging)
- [Lychee anchor checks](https://lychee.cli.rs/recipes/anchors/)
- [Lychee root-directory handling](https://lychee.cli.rs/recipes/root-dir/)
