---
layout: post
title: "Omarchy Super+K on macOS: the cheat sheet is the feature"
date: 2026-10-22
categories: macos productivity
---

Everyone ports Omarchy's keys to macOS. Nobody ports the menu that teaches them.

Omarchy's `Super+K` isn't a launcher — it's a cheat sheet. It pipes `hyprctl binds` into Walker: floating window, fuzzy filter, `Key -> action` aligned in columns. The keymap is the speed layer; `Super+K` is the discovery layer. Without it you have fast keys nobody can learn.

I rebuilt it on a Mac in an afternoon. The stack: AeroSpace (tiling, Super = Cmd), Raycast (launcher), and a real Raycast extension that parses the live AeroSpace config.

## The keymap (AeroSpace)

`~/.config/aerospace/aerospace.toml`, `config-version = 2`:

```toml
cmd-enter = 'exec-and-forget open -a Ghostty'
cmd-space = 'exec-and-forget open -a Raycast'   # later moved, see below
cmd-1 ... cmd-0 = 'workspace 1 ... 10'
cmd-shift-1 ... = 'move-node-to-workspace ...'
cmd-h/j/k/l = 'focus ...'
cmd-k = 'exec-and-forget open -g raycast://extensions/ankitg12/omarchy-macos-keybindings/index'
```

Two things broke on the way. My first config had `cmd-j` bound twice (focus-down and join) — AeroSpace refused the whole file with a 400 and no window moved until I read the actual error. And `alt-tab = 'focus next-window'` is wrong syntax; it's `focus dfs-next`. Both caught by `aerospace reload-config` exit code, not by guessing.

## Why a script command wasn't enough

First attempt: a Raycast script command (`\@raycast.mode fullOutput`) that printed the binds as colored text. It worked. It was also wrong — `fullOutput` is display-only. You can read it, you can't filter it. Omarchy's menu narrows as you type; a static dump doesn't. Dead end, deleted the script.

The real thing is a TypeScript `List`: sections per bind group, `pretty` keys (`⌘ ⇧ F`), `Enter` to copy the action. Same TOML parser, but inside `@raycast/api` so the list is live-filtered:

```tsx
<List searchBarPlaceholder="Filter keybindings… (Omarchy Super+K)">
  {sections.map(([section, items]) => (
    <List.Section key={section} title={section}>
      {items.map((b) => (
        <List.Item key={b.key} title={b.pretty} subtitle={b.command} />
      ))}
    </List.Section>
  ))}
</List>
```

Repo: [raycast-extensions](https://github.com/ankitg12/raycast-extensions), flat layout, one dir per extension — future ones need not be Omarchy-related.

## The Spotlight decision

`Cmd+Space` opened both Raycast and Spotlight. I gave it back to Spotlight and moved Raycast to `Cmd+Opt+Space`. Rationale: Spotlight owns system search (files, math, clipboard-adjacent lookups); Raycast owns commands. One chord per tool, no overlap. Omarchy gets away with one launcher because Walker does both; on macOS the split is cleaner.

## What I stole, what I skipped

[PSA Labs' Omarchy-on-macOS guide](https://psalabs.eu/articles/2026/08/omarchy-hyprland-workflow-on-macos/) is the best prior art — Option-based binds, JankyBorders, SwipeAeroSpace trackpad swipes, dotfiles bootstrap. They ship no `Super+K` equivalent; their keybindings live in static docs. That's the gap this post fills: parse the live config, never let the docs drift.

Skipped for now: JankyBorders (focused-window border), SwipeAeroSpace (three-finger workspace swipes). Both worth adding; neither changes the `Cmd+K` story.

## Source

- [raycast-extensions](https://github.com/ankitg12/raycast-extensions) (MIT) — the `keybindings` extension
- [AeroSpace guide](https://nikitabobko.github.io/AeroSpace/guide) — `config-version = 2` required, `dfs-next` syntax
- [Omarchy hotkeys](https://omarchy.org/manual/hotkeys/) — what `Super+K` actually runs
- [Raycast: create your first extension](https://developers.raycast.com/basics/create-your-first-extension) — `npm run dev` bypasses lint; lint only gates Store publish
