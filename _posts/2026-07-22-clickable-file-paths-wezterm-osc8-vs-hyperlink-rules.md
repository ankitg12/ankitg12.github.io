---
layout: post
title: "Clickable file paths in your terminal: why they work in your AI agent but not your shell"
date: 2026-07-22
categories: wezterm terminal tools productivity
series: "AI coding agent productivity"
---

My AI coding agent prints a file path and I click it — VS Code opens the file. I run `ls` in the same terminal, it prints the same path, and clicking does nothing. Same terminal, same path, different outcome. That gap turns out to be the entire story of how terminals make text clickable, and closing it takes two mechanisms that are easy to confuse.

## Two mechanisms, one visual result

There are exactly two ways a path becomes clickable in a terminal, and they are not the same thing:

| | OSC 8 | `hyperlink_rules` |
|---|---|---|
| Who acts | the **program** writes escape bytes into its output | the **terminal**, at render time |
| Where it lives | in the byte stream — survives pipes, `tee`, logs, `ssh` | only in the terminal's screen model |
| Form | `ESC ]8;;file://… ESC \ text ESC ]8;; ESC \` | an internal regex overlay on screen cells |
| Scope | only output from programs that choose to emit it | **any** text on screen, from any command |
| Ambiguity | none — the producer **declares** the exact target | the terminal must **guess** where the path ends |

That last row is the whole game. **OSC 8 declares. Regex guesses.**

My coding agent (OMP) uses OSC 8: it knows the exact path it's printing, so it wraps it in an [OSC 8 hyperlink](https://gist.github.com/egmontkob/eb114294efdcd5adb1944c9f3cb5feda) escape sequence before the bytes ever hit the terminal. Node's `url.pathToFileURL(p).href` does the encoding. Raw shell output (`ls`, `git`, `find`, build logs) emits no such thing — a path there is just plain text bytes. For those, the terminal's only route to clickability is `hyperlink_rules`: regexes it runs over the screen to auto-detect links.

WezTerm's default `hyperlink_rules` match URLs (`http`, `mailto`, …) but **never bare file paths**. That's the gap.

## Fix part 1: hyperlink_rules for any command

`hyperlink_rules` is the universal lever precisely *because* it doesn't need the program to cooperate — it works on whatever is already on screen. Start from the defaults (so URLs keep working) and add file-path rules:

```lua
config.hyperlink_rules = wezterm.default_hyperlink_rules()
table.insert(config.hyperlink_rules, {
  regex  = [[[A-Za-z]:[\\/][^\s"'<>|]*]],   -- C:\Users\... or C:/Users/...
  format = 'file://$0',
})
```

This works — until a path has a space. The `[^\s…]` stops at the first whitespace, so `C:\OneDrive - AMD Inc\file.md` links only as far as `C:\OneDrive`.

## Test the regex against the *same engine* the terminal uses

Before editing the live config, you can test WezTerm regexes for free: WezTerm uses the **Rust `regex` crate**, and so does `ripgrep`. So `rg -o` shows you the exact span WezTerm will linkify:

```bash
rg -o -N -e '[A-Za-z]:[\\/][^\s"'"'"'<>|]*' paths.txt
```

`-o` prints only the matched span — which is exactly the clickable region. This turns "edit config, reload, squint at the screen" into a tight unit-test loop. One caveat it also surfaces: the Rust crate has **no lookahead**, so any pattern you design has to live within that.

## Handling spaces: anchor on the extension, not on whitespace

The insight that beats the space problem: a file path runs *until its extension*, not until the next space. Anchor the regex on `.ext` at a word boundary and interior spaces become legal:

```lua
config.hyperlink_rules = wezterm.default_hyperlink_rules()
-- Rule 1: extension-anchored — interior spaces allowed.
table.insert(config.hyperlink_rules, {
  regex  = [[(?:[A-Za-z]:|~)[\\/][^\r\n"'<>|]*\.[A-Za-z0-9]{1,8}\b]],
  format = 'file://$0',
})
-- Rule 2: whitespace-stop fallback — extensionless dirs.
table.insert(config.hyperlink_rules, {
  regex  = [[(?:[A-Za-z]:|~)[\\/][^\s"'<>|]*]],
  format = 'file://$0',
})
```

Verified with `rg -o` against a corpus:

| Input | Result |
|---|---|
| `C:\OneDrive - AMD Inc\file.md` | full path ✓ (spaces handled) |
| `C:\Program Files\Some App\config.json` | full path ✓ |
| `C:\Users\me\.config\nvim\init.lua` | full ✓ (doesn't stop at `.config`) |
| `C:\Users\me\projects` (extensionless dir) | ✓ via rule 2 |
| `C:\Program Files` (spaced **dir**) | ⚠ partial — `C:\Program` only |

Rule 1 must come first: on an overlap, WezTerm takes the earlier rule, so spaced *files* win over the whitespace-stop fallback.

The click needs somewhere to go. A single `open-uri` handler routes any `file://` link to the editor:

```lua
wezterm.on('open-uri', function(_w, _p, uri)
  local path = uri:match('^file:///?(.+)$')
  if not path then return end                    -- not a file link → default handler
  path = path:gsub('%%(%x%x)', function(h) return string.char(tonumber(h,16)) end)
  if path:match('^~') then path = home .. path:sub(2) end
  path = path:gsub('/', '\\')
  wezterm.background_child_process({ 'cmd.exe', '/c', 'code', path })
  return true
end)
```

## The residual, and why regex can't fix it

One case stays broken: a **spaced, extensionless directory** — the exact thing your prompt shows, `PS C:\Users\me\Process Dashboard>`. Rule 1 needs an extension; rule 2 stops at the space. And no regex can fix this, because in raw text `C:\Users\me\Process Dashboard and then...` is *genuinely* ambiguous — there is no signal for where the directory name ends and the prose begins. The terminal is guessing, and it has nothing to guess with.

This is the wall that separates "guess" from "declare." To get past it you have to stop guessing.

## Fix part 2: declare the prompt's path with OSC 8

The prompt is output *you* control — so emit OSC 8 there, the same way the coding agent does. `.NET`'s `[uri]` gives you `pathToFileURL`-equivalent encoding (spaces → `%20`):

```powershell
function prompt {
    $gt = '>' * ($nestedPromptLevel + 1)
    $p  = $executionContext.SessionState.Path.CurrentLocation
    if ($env:WEZTERM_PANE -and $p.Provider.Name -eq 'FileSystem') {
        $e   = [char]27
        $uri = ([uri]$p.ProviderPath).AbsoluteUri   # file:///C:/Users/me/Process%20Dashboard
        return "PS $e]8;;$uri$e\$p$e]8;;$e\$gt "
    }
    return "PS $p$gt "
}
```

Two guards matter. `$env:WEZTERM_PANE` (set only inside WezTerm) means other terminals get a plain prompt with no stray escape bytes. `$p.Provider.Name -eq 'FileSystem'` skips non-filesystem locations like `HKLM:\` where a `file://` URI makes no sense. Now the whole `Process Dashboard` cwd is one clickable link — spaces and all — because the producer declared the exact target instead of leaving the terminal to guess.

## The model worth keeping

> **`hyperlink_rules` = universal but space-blind. OSC 8 = per-command but exact.**
>
> Regex works on any command's output without cooperation, but must infer boundaries — so it fumbles spaces and extensionless directories. OSC 8 is flawless — spaces, directories, survives pipes and `ssh` — but only for output whose producer you can make emit it.

Use the regex as the always-on baseline for the whole terminal; reach for OSC 8 on the specific surfaces you own (your prompt, a wrapper around a command you run constantly).

## Don't reinvent it where you don't have to

If you want OSC 8 for an arbitrary command without writing a wrapper, [`mash/osc8wrap`](https://github.com/mash/osc8wrap) is a pipe filter: `somecmd | osc8wrap`. I skipped it here because the two surfaces I actually cared about — every command's output (regex) and my prompt (a function I already control) — needed no new dependency. But for a one-off "make *this* command's paths exact," it's the boring, correct tool.

## Source

- WezTerm [`hyperlink_rules`](https://wezterm.org/config/lua/config/hyperlink_rules.html) and the [`open-uri` event](https://wezterm.org/config/lua/window-events/open-uri.html)
- [OSC 8 hyperlink spec](https://gist.github.com/egmontkob/eb114294efdcd5adb1944c9f3cb5feda) (Egmont Koblinger)
- Rust [`regex`](https://docs.rs/regex/) crate — the engine behind both WezTerm and ripgrep
- [`mash/osc8wrap`](https://github.com/mash/osc8wrap) — OSC 8 pipe filter
