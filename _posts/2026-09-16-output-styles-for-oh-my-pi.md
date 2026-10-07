---
layout: post
title: "Output styles for Oh My Pi: change how the agent works with you, in one markdown file"
date: 2026-09-16 17:10:00 +0530
categories: ai agents productivity
series: "AI coding agent productivity"
---

A coding agent has one way of working with you, and it is not always the way you need. Some days I want it to do the work and teach me while it does. On a lab testbed I want the opposite: I type the commands, and the agent checks each one. Claude Code has a feature for this, called *output styles*. [Oh My Pi (OMP)](https://github.com/can1357/oh-my-pi), the harness I use, did not have one, so I added it as an extension.

## Where the idea came from

Claude Code ships two styles as a plugin, [`learning-output-style`](https://github.com/anthropics/claude-code/tree/main/plugins/learning-output-style), and documents the mechanism in [Output styles](https://code.claude.com/docs/en/output-styles):

- **Explanatory:** the agent does the task and adds short `★ Insight` notes about why it chose an approach.
- **Learning:** the agent builds the structure, then leaves `TODO(human)` markers where I write the 5–10 lines that decide the design.

The plugin works through a session-start hook that adds a block of instructions to the system prompt. That told me what OMP needed: a hook that runs before each agent turn and can append text to the system prompt.

Before I wrote any code, I searched for prior art across Claude Code, Pi, and OMP. The closest match was a skill in [PoorRican/dotfiles](https://github.com/PoorRican/dotfiles/tree/master/configs/agent-skills/default/skills/autonomous-ai-agents/coding-agent-output-styles) that ports the Claude prompts and includes a Pi/OMP extension template. I used it as a reference for the hook shape.

## The design

OMP extensions can subscribe to `before_agent_start` and return text to append to the system prompt. The extension is about 200 lines of TypeScript:

```typescript
pi.on("before_agent_start", async (event, ctx) => {
  const available = getAvailableStyles(ctx.cwd);
  const active = readStyle(available);              // ~/.omp/agent/output-style.json
  const systemPromptAppend = loadStylePrompt(active, available);
  if (!systemPromptAppend) return undefined;        // style "off": nothing added
  return { systemPromptAppend, systemPrompt: appendPrompt(event.systemPrompt, systemPromptAppend) };
});
```

Four decisions shaped it:

| Decision | Reason |
|---|---|
| A style is a markdown file, not code | My first version registered a command for each style. Adding a style meant a code change, which is the wrong cost for a block of text. Now the extension scans three directories and every `<name>.md` file becomes a style. |
| Read the state on every turn | `/style <name>` writes one JSON file. The hook reads it again before each turn, so a switch applies on the next message, with no restart. |
| `off` costs zero tokens | When no style is active, the hook returns `undefined` and the request is the same as without the extension. |
| Show the style in the footer | A style changes how the agent behaves, so I want to see which one is on. The status line shows `📝 <name>`. |

Styles are found in this order, and a later file with the same name wins:

```text
<extension>/prompts/            # styles that ship with the extension
~/.omp/agent/output-styles/     # my own styles and overrides
<project>/.omp/output-styles/   # styles for one repository
```

```text
/style                  # list styles and show the active one
/style <name>           # switch
/style-off              # stop injecting; 0 extra tokens
/learning, /explanatory # shortcuts
```

## A style I wrote for my own work: `manual`

Much of my work is test execution on lab hardware. On those days I want to type the configuration commands myself, and I want the agent to do three things:

- read the system state after each of my steps and compare it with what I expected;
- tell me at once when I make a syntax error, target the wrong interface, or skip a prerequisite;
- not run state-changing commands unless I ask.

When the state does not match, the style asks for three lines: **Expected**, **Observed**, and **Remediation** with the exact command. It also keeps a running ledger of the verified steps, so that at any point I can see what is proved and what is not:

```text
Step [N]: [Action description]
- Target: [Device / Interface / Config path]
- Verification: [Command / File inspected]
- State: [PASS | FAIL | PENDING]
```

Each reply ends with the next manual step and the output I should expect from it.

## What it costs

| State | Extra tokens per turn |
|---|---|
| `off` | 0 |
| `explanatory` (1,193 characters) | about 300 |
| `learning` (3,209 characters) | about 800 |

Token counts are estimates at about four characters per token.

## Checklist for your own styles

- Write the style as instructions about *how to work and report*, not about the task. Task rules belong in the project's agent instructions.
- Start with `off` as the default, so that the extension costs nothing until you choose a style.
- Drop the file into `~/.omp/agent/output-styles/` and run `/style <name>`. No restart and no code change.

## Update, 7 October 2026: a `ledger` style

After three weeks of use, I wanted the agent to run commands itself on most days, but I still wanted the step ledger from `manual`. Those are two separate wishes, so I made them two separate styles. `ledger` keeps only the reporting rules, and the normal confirmation rules of the harness decide who executes:

```text
Step [N]: [Action description]
- Target: [Device / Interface / Repo / Config path]
- Verification: [Command run / File inspected]
- State: [PASS | FAIL | PENDING]
```

| Rule | Why |
|---|---|
| A row is PASS only when its verification output is in this session | A result from memory or from an earlier session is a claim, not a proof. PENDING is the correct state for it. |
| Show only PENDING, FAIL, and changed rows in full; fold the other PASS rows into one line | If every reply repeats every row, reply *n* carries *n* rows, and a session of *N* replies costs about *N²/2* rows. |
| End with the next step and its expected output | I can check the result against a stated expectation, not against my impression. |
| No ledger for a reply that ran no tools | A ledger under a conversational reply is noise. |

The split follows a principle that Edsger Dijkstra named in 1974 in [*On the role of scientific thought* (EWD447)](https://www.cs.utexas.edu/~EWD/transcriptions/EWD04xx/EWD447.html): *the separation of concerns*. A style file is a module of the prompt. `manual` controls two things, who executes and how the work is reported. `ledger` controls only the second. Each switch should change exactly one behaviour.

`ledger` is 1,125 characters, about 280 tokens per turn. It also adds output tokens to each reply, and folding the settled rows stops that cost from growing with the length of the session.

Three items for the checklist:

- Give each style one concern. If its description needs the word "and", split it.
- If a format repeats in every reply, work out how its size grows over a long session.
- Commit the styles you reuse to the repository that ships the extension. Keep the user directory for local overrides.

## Source

- [`omp-output-styles` extension](https://github.com/ankitg12/omp-extensions/tree/main/packages/omp-output-styles)
- [Oh My Pi](https://github.com/can1357/oh-my-pi)
- [Claude Code `learning-output-style` plugin](https://github.com/anthropics/claude-code/tree/main/plugins/learning-output-style) and [Output styles docs](https://code.claude.com/docs/en/output-styles)
- [PoorRican/dotfiles: coding-agent-output-styles skill](https://github.com/PoorRican/dotfiles/tree/master/configs/agent-skills/default/skills/autonomous-ai-agents/coding-agent-output-styles)
- E. W. Dijkstra, [*On the role of scientific thought*, EWD447](https://www.cs.utexas.edu/~EWD/transcriptions/EWD04xx/EWD447.html), 30 August 1974
