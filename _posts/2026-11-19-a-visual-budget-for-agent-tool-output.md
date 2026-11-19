---
layout: post
title: "A Visual Budget for Agent Tool Output"
date: 2026-11-19
categories: ai agents productivity
---

A 30-line tool result can still fill a screen when each line wraps several times.

I hit this while using [Oh My Pi](https://github.com/can1357/oh-my-pi) (OMP), a coding agent. My [previous post]({% post_url 2026-11-14-tool-output-belongs-in-an-artifact %}) covered an extension that limits tool output and saves larger results as artifacts. The limit counted newline-delimited lines. That kept the result small in one sense, but it did not bound what I saw in the terminal.

A line with 1,000 characters might take ten screen rows. A 30-line preview made of such lines could therefore take far more than 30 rows. This is an interface problem as well as a context problem: I want to see what the agent is reading without losing the full result.

## Three ways to handle a long line

The [extension](https://github.com/ankitg12/omp-extensions/tree/main/packages/tool-output-lines) now has three long-line policies:

| Policy | What the preview favors | Cost |
|---|---|---|
| Preserve | Existing newline-based behavior | A long line can use many screen rows |
| Wrap | More of the original text | Fewer source lines fit in the budget |
| Truncate | More source lines in the preview | The ends of long lines are hidden |

The visual budget includes the output notice. When the extension shortens a result, it saves the received output as a `local://` artifact and includes a reference in the preview. That artifact contains what the extension received; an upstream tool may already have shortened its result.

An explicit read range has a useful property: it can give a few long lines more room. In truncate mode, the extension uses the wrap budget for an explicit range by default. If the selected text fits, the agent gets the full line without disabling the limit. If a single line exceeds the entire budget, a line range alone is not enough to recover it; character slicing is still needed.

## Let the agent choose, within the human's limit

Reading code and scanning a list have different needs. I added an optional choice for individual reads: append `:wrap` or `:truncate` to the existing read path. The extension removes the suffix before the native read and applies the selected preview policy to the result.

```text
read source.ts:10-20:wrap
read source.ts:10-20:truncate
```

The human can enable or disable this choice with `allowAgentMode`. It is off by default. When enabled, the agent receives the syntax in its instructions. The extension also reports the effective settings when they change, including the line and character limits, width, policy, and permission. The agent can choose how to use the budget, but not increase the configured budget through these selectors.

The configuration is read as results arrive, so a settings change does not require an extension reload. The code change itself does. After a reload, I tested the same long line with both selectors: wrap returned it in full, while truncate shortened it and returned an artifact reference.

## Characters are not terminal columns

Counting Unicode characters is not enough for a visual estimate. In one local check, 80 ASCII characters occupied 80 terminal cells, while 80 CJK characters occupied 160. The extension uses [Bun's `stringWidth`](https://bun.com/docs/runtime/utils#bun-stringwidth) to estimate cell width, and follows OMP's three-cell tab convention. The [wcwidth documentation](https://wcwidth.readthedocs.io/en/stable/intro.html) explains why wide characters and control sequences matter.

This is still an estimate, not a guarantee of an exact screen height. The terminal can be narrower than the configured width; line-number gutters, control sequences, and renderer behavior can also affect the result. Very narrow budgets can leave too little space even for an artifact notice. Streaming output can appear before the final-result limit applies.

Line-width discipline helps with code we control, but it is not a substitute for bounded previews. The [Linux kernel coding style](https://www.kernel.org/doc/html/latest/process/coding-style.html) prefers 80 columns while allowing longer lines when splitting them would hurt readability. Logs, external documents, and generated data do not necessarily follow that discipline.

## Others are working on the same edges

| Prior work | Relevant idea |
|---|---|
| [OMP issue #7081](https://github.com/can1357/oh-my-pi/issues/7081) | Truncate collapsed output and wrap expanded output. This is a user-view choice, not a per-read agent choice. |
| [Pi's tool-result truncation](https://github.com/earendil-works/pi/issues/134) | Combine line and byte limits, preserve full output, and tell the agent how to continue. |
| [Claude Code issue #41092](https://github.com/anthropics/claude-code/issues/41092) | Line-based reads can fail to recover an oversized line. |
| [Pydantic AI tool output limits](https://pydantic.dev/docs/ai/harness/tool-output-limits/) | Offer strategies such as truncation and retaining large results outside the immediate context. |
| [MCP pagination](https://modelcontextprotocol.io/specification/2025-11-25/server/utilities/pagination) | Use a continuation cursor for large lists; this does not by itself solve a single long line. |

One [study of agent pagination](https://arxiv.org/abs/2608.26130) reports that agents in its observed logs did not request second pages. That is a reason to make the first preview useful and its omissions visible, rather than assuming the agent will always follow a continuation.

The aim is not to hide output. It is to keep the first view small enough for a person to inspect, while giving the agent an explicit way to trade breadth for detail.

## Source

- [Tool-output extension](https://github.com/ankitg12/omp-extensions/tree/main/packages/tool-output-lines)
- [OMP read display discussion](https://github.com/can1357/oh-my-pi/issues/7081)
- [Pi tool-result truncation](https://github.com/earendil-works/pi/issues/134)
- [Agent pagination study](https://arxiv.org/abs/2608.26130)
