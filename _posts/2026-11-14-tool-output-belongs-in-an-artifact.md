---
layout: post
title: "Tool Output Belongs in an Artifact, Not the Whole Context"
date: 2026-11-14
categories: ai agents productivity
---

A directory listing filled the terminal and sent far more detail to the agent than the task needed. The next question was simple: could the agent see a short preview, then request a specific part of the full output when it needed more?

I built a [tool-output extension](https://github.com/ankitg12/omp-extensions/tree/main/packages/tool-output-lines) for [oh-my-pi](https://github.com/can1357/oh-my-pi), the coding agent I use. The interesting part was not cutting text at a line count. It was deciding where the complete result lives, how the agent finds it again, and whether the human sees the same preview as the model.

## Start with a limit, not a summary

The first design kept ten lines per tool result. The last line was a notice; the other nine were the beginning of the result. The complete text was saved before the preview replaced it. The agent could read a selected range from the saved copy.

This is progressive disclosure. A preview leaves detail in the artifact, where a targeted read can retrieve it. The line limit reduces what enters the model's context. The original command still performs the same work. [Vercel's tool-output design guide](https://vercel.com/academy/build-ai-agent-harness/tool-output-design) makes the complementary point: tools should produce bounded, structured output upstream. This extension is a safety net for results that are still large.

A separate character limit handles a different case: a single very long line. Both limits are configurable, with overrides for individual tools. For example:

```json
{
  "maxLines": 30,
  "maxPreviewChars": 4000,
  "tools": {
    "bash": { "maxLines": 20 }
  }
}
```

The limit includes the notice. A tool-specific setting inherits any global field it does not override. Ten lines proved restrictive for this workflow: I had to request many small ranges. I changed the local setting to 20 and then to 30 as a trial; there is no claim that 30 is optimal for every task or terminal.

## A path is an interface

The first notice pointed to an absolute filesystem path. It preserved the output, but the path wrapped across several terminal lines and dominated the notice. That defeated the point of a compact preview.

The next version uses the agent's native artifact reference:

```text
[103 lines · full output: artifact://122]
```

The number identifies an artifact in the current session. An agent can use `read` with a range such as `artifact://122:30-40:raw` to retrieve more. In a live test, a command that produced 100 numbered lines returned 29 lines and a notice at the 30-line setting. A later range read retrieved the remaining detail.

There is an important failure boundary: if the extension cannot save the artifact, it leaves the result unchanged. A short preview is not worth losing the evidence. The saved artifact contains the result received by the extension; any truncation performed earlier by the tool has already happened.

## The terminal and the model saw different results

A screenshot exposed a defect in the first implementation. The model received a bounded result, but a `read` card in the terminal still displayed the longer original text. The reason was a second representation: the terminal renderer preferred structured display content in the result metadata, while the extension had changed only the model-facing text.

I changed the extension to bound structured previews too. An artifact read also displayed its resolved on-disk path, which made the compact reference look long again; the bounded artifact preview now suppresses that redundant path. An `eval` cell had the same split between model text and terminal metadata. An audit found similar structured output in `glob`, AST search and edit, and task results.

The renderer test used a 100-line result and a 30-line limit. It confirmed that line 29 and the artifact reference were visible, line 30 was not, and the resolved-path notice was absent. This distinction matters: limiting the model's context does not automatically make the human interface concise. A live subagent test also saw line 29 and the artifact notice. OMP normally passes loaded extensions to subagents; an isolated subagent that disables extensions is an exception. Streaming output can appear before the final-result limit applies.

## Put configuration where it belongs

The extension reads its configuration from `$XDG_CONFIG_HOME/omp/tool-output-lines.json`. If `XDG_CONFIG_HOME` is unset, empty, or not an absolute path, it uses `~/.config/omp/tool-output-lines.json`, following the [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir/latest/).

The configuration is read for each tool result, so changes take effect on the next result. Other extensions still use their existing configuration paths; migrating those consistently is a separate task.

## What the exercise changed

| Question | Decision |
|---|---|
| How much enters context? | A configurable line and character preview |
| Where is the rest? | A session artifact, saved before shortening |
| How does the agent retrieve it? | A short reference and a targeted range read |
| What does the human see? | A bounded preview for supported structured tool results |
| What if saving fails? | Keep the complete result unchanged |

The result is a smaller default surface that preserves the tool result. The design depends on the full result remaining available and on the terminal and model views agreeing about what was shortened.

## Source

- [oh-my-pi](https://github.com/can1357/oh-my-pi)
- [Tool-output extension](https://github.com/ankitg12/omp-extensions/tree/main/packages/tool-output-lines)
- [Tool Output Design, Vercel Academy](https://vercel.com/academy/build-ai-agent-harness/tool-output-design)
- [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir/latest/)
