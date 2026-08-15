---
layout: post
title: "The Second Copy: Why Reusing Your Agent's Internal UI Crashes It"
date: 2026-08-15 12:00:00 +0530
categories: ai agents productivity
series: "AI coding agent productivity"
---

My extension imported a function from the coding agent hosting it, called that function, and
killed the agent outright:

```
[Uncaught Exception] TypeError: undefined is not an object (evaluating 'theme.fg')
    at renderPauseScreen (…/src/modes/components/pause-screen.ts:102:28)
```

The function works. The agent runs it every time you type `/pause`. It only explodes when *my*
extension is the caller.

Everything below was observed on one host at one version — OMP `17.3.1`, running under Bun. I
have not reproduced it under esbuild or webpack, so treat the mechanism as a hypothesis worth
checking against your own bundler rather than a property of bundlers in general. The check is
cheap, and the failure is not.

## The setup

I had built a keyboard shortcut that pauses every background agent during a scheduled break.
My first version drew its own pause screen: my box-drawing, my clock, my resume keys. It worked
and it looked like a forgery. The host already had a beautiful pause screen behind its `/pause`
command, and the reviewer's objection was the correct one — *why are you painting a second one?*

So I deleted my version and imported the real one:

```ts
import { runPauseScreen } from "@oh-my-pi/pi-coding-agent/modes/components/pause-screen";
```

That import resolves. The module loads. And the first call takes the whole process down.

## What actually happens

The agent ships as a **bundle** — one `dist/cli.js` with every module folded in and every
singleton initialised at startup. My extension is a loose `.ts` file, loaded later by a plain
dynamic import.

When I import a deep source path, I do not reach into the running bundle. I load
`src/modes/components/pause-screen.ts` **fresh from disk**. And a source file is not
self-contained — it pulls in its own neighbours:

```ts
// inside pause-screen.ts
import { theme } from "../theme/theme";                      // relative → fresh copy
import { agentPauseGate } from "@oh-my-pi/pi-agent-core";     // bare → resolved by the host
```

Those two lines behave completely differently, and that is the entire lesson.

| Specifier | Example | What you get |
|---|---|---|
| **Bare package** | `@oh-my-pi/pi-agent-core` | Remapped by the host's resolver plugin to the **running bundle** — shared state |
| **Relative path** | `../theme/theme` | A **brand-new module instance**, loaded from disk, initialised by nobody |

The host installs a Bun plugin precisely so bare specifiers resolve to itself:

```ts
Bun.plugin({
  name: "omp:legacy-pi-shim",
  setup(build) {
    build.onResolve({ filter: LEGACY_PI_SPECIFIER_FILTER }, resolveLegacyPiSpecifier);
  },
});
```

Relative imports never touch that plugin. So `agentPauseGate` was the real, shared gate — while
`theme` was a second, pristine, **uninitialised** copy. The bundle's theme had been populated at
startup; mine had never been told anything. `theme.fg` was `undefined`, and the crash was not in
my code at all. It was in the host's own well-tested rendering function, executing against a
singleton that only my copy could see.

This is the classic **dual-instantiation** failure — the same disease as two copies of React in
one page, or two `pg` pools when a transitive dep pins a different minor. The novelty is only
that the boundary here is *bundle versus source*, not `node_modules` layout, so no dependency
tree will show it to you.

## The fix, and its price

The host hands the live objects to any custom UI component it renders. So take the real theme
from the host and pour it into the duplicate:

```ts
await ctx.ui.custom<void>((tui, theme, _keybindings, done) => {
  setThemeInstance(theme);          // seed MY copy with the host's live instance
  const host = { ui: tui, showStatus: (m: string) => ctx.ui.notify(m) };
  void runPauseScreen(host).then(() => done(undefined));
  return new InvisibleHostComponent();
});
```

That works. The native screen renders, the clock ticks, the native resume keys resume.

It is also a **seam, not a foundation**, and I want to be honest about that rather than dress it
up as a pattern. I am reaching past a public API into a private one, and holding it together by
hand-initialising a singleton the author never expected a stranger to touch. Any refactor that
adds one more relative import with state moves this seam without warning. The durable version is
a small capability on the extension API — six lines in the host — and that is what I would
upstream.

## How to know before it bites you

Three checks, in order:

1. **Audit the target file's imports.** Every *relative* import is a fresh copy for you. Ask of
   each one: does it hold state that someone initialised at startup? Those are your landmines.
   Bare package specifiers are usually fine — but verify rather than assume; that is a claim
   about your host's resolver, not a law of nature.
2. **Never trust a type-only import as proof of anything.** My extension had imported from this
   package for weeks with no trouble, because those imports were `import type` and vanished at
   compile time. The first *value* import is the first real test.
3. **Test in a disposable process, not your own.** Extensions load at process start, so a
   reload command tests the *old* code while you read the new code and congratulate yourself.
   Spawn a throwaway agent, drive it, and read its raw pane. When it dies, the stack trace is
   in the pane — the agent-level query returns empty, because there is no agent any more.

That third point earned its place the hard way. I nearly asked for a reload and reported the
result of code that was not running.

## The part that is not about JavaScript

The crash was the cheap half of this. The expensive half was that I twice declared a detector
"broken" without capturing its raw output, and once declared a requirement "undocumented" after
searching two directories out of four — when it was written down plainly in a post I had
published the day before.

Both errors share a shape: **highest confidence exactly where I had looked least.** A stack
trace forces you to be right. An absence never does. If you take one habit from this, take that
one — before you say a thing is not written down anywhere, name the surfaces you searched, and
notice how few they are.

## Source

- [Building attention-pause for Herdr and OMP]({% post_url 2026-08-14-building-attention-pause-for-herdr-and-omp %}) — the plugin this extension serves
- [Bun plugin API — `onResolve`](https://bun.sh/docs/bundler/plugins)
- [Node.js — package `exports` and subpath patterns](https://nodejs.org/api/packages.html#subpath-patterns)
- [Testing TUIs with tmux](https://ghassan-alhamoud.com/articles/testing-tuis-with-tmux.html) — a stricter version of the disposable-process discipline
