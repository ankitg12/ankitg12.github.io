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

## Zoom out, then zoom in

A head-only preview makes the beginning easy to inspect and the rest invisible. I added an experimental `stride` mode that selects sample points across the result. Each point can include surrounding lines, like [grep's `-B` and `-A` context options](https://man7.org/linux/man-pages/man1/grep.1.html). Context is intended to fit the same line budget: more context means fewer sample points; if two points with their context cannot fit, the implementation drops the context. The minimum-budget edge case below needs a fix.

The last output line is still a notice. It says the result is sampled and non-contiguous and links to the saved result. A sample is an overview, not proof that omitted lines are unimportant. It is also possible to miss a rare event in a repetitive log.

These additions are in local commits; they are **not yet in the published extension**. The intended configuration sets the mode globally or for a named tool:

```json
{
  "maxLines": 30,
  "maxPreviewChars": 4000,
  "tools": {
    "read": { "mode": "stride", "before": 1, "after": 1 }
  }
}
```

The local version also has a session-only `/tool-lines` command. `/tool-lines stride context=1 30` selects stride mode, one line of context on each side, and a 30-line budget. `/tool-lines head` selects a head preview; `/tool-lines off` disables the extension's truncation for that session; `/tool-lines on` resets the override to the config. The command does not edit the config file.

### Line numbers are part of the contract

A read result can already be a selected range or a structural summary. Its saved artifact contains that **result**, not necessarily the original file. Artifact line numbers count positions in the saved result; source line numbers refer to the file. The updated notice distinguishes these coordinates. For a source file, use the source path and a source range; for a command result, use a range of the saved artifact.

A fresh-session test found a remaining defect: when a raw ranged read began at source line 35, stride labelled the first sampled line `1`. Until that is fixed, the displayed numbers from sampled raw ranged reads are not reliable source coordinates. This is a reason to verify a source range against the original file before acting on it. Code inspection also shows that at the minimum `maxLines: 2`, stride keeps two sample lines plus the notice, exceeding the intended two-line budget. Both defects remain open in the local version.

A second mismatch appeared in skill reads. The read tool adds a provenance line before its numbered text. The extension originally checked only the first line for a number, so it saved the remaining prefixes; reopening that artifact added another set of numbers. A local fix handles the provenance wrapper and preserves raw content that genuinely starts with `1|`. Existing artifacts retain the bytes they were saved with.

### Existing tools and prior art

| Tool | What it contributes | Fit for this extension |
|---|---|---|
| [sed](https://man7.org/linux/man-pages/man1/sed.1.html) and [awk](https://man7.org/linux/man-pages/man1/awk.1p.html) | Select lines at regular intervals. | Useful in a shell pipeline; the OMP hook still needs to transform arbitrary tool results. |
| [grep](https://man7.org/linux/man-pages/man1/grep.1.html) | Keeps lines around a match with `-B` and `-A`. | A useful model for context, but it selects around matches, not evenly spaced points. |
| [VenetianBlinds.js](https://github.com/firasd/vblinds) | Samples evenly spaced character positions and shows context. | Closest prior art. Its character windows can cut through lines, so it does not directly meet a whole-line output budget. |
| [Aider's repository map](https://aider.chat/docs/repomap.html) | Selects code symbols within a token budget. | A code-aware overview, rather than a sample of arbitrary text. |
| [ast-grep outline](https://ast-grep.github.io/guide/outline-code) and [Tree-sitter code navigation](https://tree-sitter.github.io/tree-sitter/4-code-navigation.html) | Find code structure and source positions. | Useful for choosing a function to inspect; they do not replace sampling logs or other text. |
| OMP's native read summary | Shows code declarations and ranges before a detailed read. | An existing code-aware zoom-out; applying stride afterward can remove declarations from that summary. |

A local test with ast-grep 0.51.3 returned a function's source range, which could then be read directly. OMP's embedded ast-grep was 0.50.0 and did not support `outline`; in this installation, setting `PI_DISABLE_UUTILS_BUILTINS=1` let a fresh OMP session use the locally installed executable. That switch affects the embedded utility replacements as a group, not only ast-grep. I used `outline` independently to inspect the extension's exported symbols; the output-limiting extension does not call it or depend on it. An agent can use `ast-grep outline path/to/file.ts --items exports` to select a symbol, then `read` the relevant source range. OMP's own code summary is another option for the same initial zoom-out.

**Why implement stride here?** The integration point is an OMP `tool_result` hook, where the result is already in memory and may come from many tools. The intended contract is a fixed line budget, whole lines, optional before/after context, a saved result, and matching model and terminal previews. `sed` and `awk` can select lines, and `grep` can add context, but using them here still leaves the hook, budget allocation, artifact handling, and display metadata to the extension. Calling a separate process for every tool result would add another moving part. VenetianBlinds provides a close sampling approach, but its context is measured in characters; its documented interface does not provide the same whole-line budget. The code-aware options solve a different problem.

The alternatives are not interchangeable. For code, a syntax-aware outline can give a more meaningful overview than every Nth line. For arbitrary tool output, a bounded sample and an artifact remain useful. No task-level comparison has yet established which approach makes an agent more effective, and the raw-range defect shows that the custom integration still has correctness work to do.

## What the exercise changed

| Question | Decision |
|---|---|
| How much enters context? | A configurable line and character preview |
| Where is the rest? | A session artifact, saved before shortening |
| How does the agent retrieve it? | A short reference and a targeted range read |
| What does the human see? | A bounded preview for supported structured tool results |
| What if saving fails? | Keep the complete result unchanged |

The result is a smaller default surface that preserves the tool result. The design depends on the full result remaining available and on the terminal and model views agreeing about what was shortened.

## Update: making the preview easier to use

A later terminal check showed that the model and the human could see different formatting for the same sampled Markdown. The model received numbered lines; the terminal rendered the text as Markdown and joined some lines together. The read preview now uses plain text, so the line numbers and breaks can be checked in the terminal too.

A short range read also exposed a limit interaction. The read tool can add context around a requested range, making its result longer than the requested range. The extension now leaves an explicit range within the configured read limit intact. Larger reads remain subject to the limit.

The read tool can shorten a result before the extension receives it. That produced two notices and two artifacts: one held the full output, while the other held the already-shortened result. When the tool's metadata confirms the full artifact, the extension now shows one combined notice and points to that artifact. A restarted-session check retrieved lines from the section that the tool had omitted. For a byte-limited read without a full artifact, the combined notice instead identifies the saved partial result and tells the agent where to continue in the source.

The extension adds range-reading guidance to the system prompt before each user prompt. When stride is enabled, the guidance also explains that a sample is not contiguous and distinguishes source line numbers from positions in a saved result. This is separate from the short notice on each shortened result.

A separate code-outline mode was removed in favour of the read tool's existing structural summary. That does not make stride a code-aware outline: sampling a long structural summary can still remove declarations, so an exact range read remains important.

A further check found that sampled raw reads needed source coordinates. Their previews are limited like other reads, while the saved artifact retains the tool output verbatim. A raw read within the limit remains unchanged. The terminal preview uses a one-based gutter for display rows, separate from any source numbers in the text. The minimum-budget edge case noted above was not revalidated in this update.

## Source

- [oh-my-pi](https://github.com/can1357/oh-my-pi)
- [Tool-output extension](https://github.com/ankitg12/omp-extensions/tree/main/packages/tool-output-lines)
- [Tool Output Design, Vercel Academy](https://vercel.com/academy/build-ai-agent-harness/tool-output-design)
- [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir/latest/)
