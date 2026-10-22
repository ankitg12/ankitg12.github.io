---
layout: post
title: "The Mac Finally Gets Ghost Text: Oh My Zsh + Autosuggest + Atuin"
date: 2026-10-22
categories: macos terminal tools
---

Windows had it. Linux had it. The Mac had neither — every shell restart meant retyping commands from memory, while PowerShell showed faded grey hints and bash showed inline completions.

The fix took one session: Oh My Zsh plus three plugins and a history daemon.

---

## The stack

| Piece | What it does | Keybinding |
|-------|--------------|------------|
| `zsh-autosuggestions` | Grey ghost text as you type, from history | `→` accepts, `Ctrl+E` line end |
| `fzf` | Fuzzy history search + file finder | `Ctrl+R` history, `Ctrl+T` files |
| `atuin` | SQLite history with directory + exit-code context | Replaces `Ctrl+R` backend |
| `fast-syntax-highlighting` | Red for typos before you hit enter | Passive |
| `sudo`, `history-substring-search`, `colored-man-pages` | OMZ builtins that earn their keep | `Esc` `Esc` for sudo prefix |

History itself: 50,000 lines, shared across tabs, duplicates dropped:

```zsh
HISTSIZE=50000
SAVEHIST=50000
setopt SHARE_HISTORY HIST_IGNORE_ALL_DUPS HIST_REDUCE_BLANKS INC_APPEND_HISTORY
```

Shell-agnostic — this is zsh config, not terminal config. Same behaviour in Tern, Ghostty, and WezTerm.

---

## Three traps on the way in

**Trap 1: Homebrew owned by someone else.** `brew install` failed because `/opt/homebrew` belonged to another user on the box. Fix is Homebrew's own recommended layout:

```bash
sudo chown -R $(whoami):admin /opt/homebrew
```

**Trap 2: a plugin fight.** `zsh-autocomplete` and the syntax highlighter both claim the `menu-search` and `recent-paths` ZLE widgets — every new shell printed:

```
zsh-syntax-highlighting: unhandled ZLE widget 'menu-search'
zsh-syntax-highlighting: unhandled ZLE widget 'recent-paths'
```

Load order (highlighter last) is the documented fix per [marlonrichert/zsh-autocomplete#711](https://github.com/marlonrichert/zsh-autocomplete/issues/711), but the warnings persisted — so `zsh-autocomplete` came out of the plugin list entirely. Its dropdown duplicates what Tab plus ghost text already give. Everything else stayed.

**Trap 3: the hook recursion.** With `history-substring-search` listed before `fast-syntax-highlighting`, every Enter printed:

```
azhw:zle-line-finish:3: maximum nested function level reached; increase FUNCNEST?
```

HSS registers a `zle-line-finish` hook first (creating the `azhw:` dispatcher); FSH loads after and wraps the dispatcher instead of the raw widget, so each line-finish re-enters the wrapper. Fix is the order printed below — highlighter before substring search. Upstream tracks the real fix in [fast-syntax-highlighting#89](https://github.com/zdharma-continuum/fast-syntax-highlighting/pull/89) (open). Verified with live commands in a running Tern tab, not just a fresh shell probe.

---

## The `.zshrc` core

```zsh
export ZSH="$HOME/.oh-my-zsh"
ZSH_THEME="robbyrussell"
plugins=(
    git
    macos
    fzf
    sudo
    colored-man-pages
    fast-syntax-highlighting
    zsh-autosuggestions
    history-substring-search
)
source $ZSH/oh-my-zsh.sh

ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE="fg=8"
ZSH_AUTOSUGGEST_STRATEGY=(history completion)
ZSH_AUTOSUGGEST_USE_ASYNC=true

source <(fzf --zsh)
eval "$(atuin init zsh)"
```

Atuin imported 1,043 existing history lines on first run (`atuin import zsh`), so day one already had suggestions. Startup cost: plugins add ~0.7s; the rest is pre-existing `nvm` load.

---

## Source

- [Oh My Zsh](https://github.com/ohmyzsh/ohmyzsh) — framework, `robbyrussell` theme
- [zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions) — ghost text
- [fast-syntax-highlighting](https://github.com/zdharma-continuum/fast-syntax-highlighting) — typo highlighting
- [fzf](https://github.com/junegunn/fzf) — fuzzy history + file search
- [Atuin](https://github.com/atuinsh/atuin) — SQLite history with context
- Guide that prompted the full install: [solomk.in/posts/terminal-configuration](https://solomk.in/posts/terminal-configuration)
- Plugin-order upstream: [zsh-autocomplete#711](https://github.com/marlonrichert/zsh-autocomplete/issues/711)
