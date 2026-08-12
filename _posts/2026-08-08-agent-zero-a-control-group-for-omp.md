---
layout: post
title: "Agent Zero: A Control Group for Your Coding Agent"
date: 2026-08-08 08:36:00 +0530
categories: ai agents productivity tools
series: "AI coding agent productivity"
---

A coding agent can consume thousands of tokens before you type the first word.

Some of that context is the harness itself. The rest is accumulated configuration: instruction files, skills, memories, extensions, MCP servers, and tool schemas. Once they are mixed together, it becomes difficult to answer a basic engineering question:

> What does this customization actually cost?

I previously described [patterns for specialized Claude and Codex agents]({% post_url 2026-03-29-patterns-of-agentic-ai %}) and then built a way to [measure an agent's static and dynamic context]({% post_url 2026-07-23-know-what-your-agent-sees %}). The missing piece was a control group: the same harness and model, with everything optional removed.

I call it **Agent Zero**.

## What Agent Zero is

Agent Zero is not another agent framework. It is an isolated [Oh My Pi (OMP)](https://github.com/can1357/oh-my-pi) launch profile with:

- no personal or project instruction files;
- no skills or extensions;
- no memory injection;
- no subagents or coordination layer;
- only the tools required for ordinary repository work and web research.

The purpose is measurement, not asceticism. Start from Zero, add one capability, and measure its marginal context cost.

This is the agent equivalent of a clean-room benchmark: change one variable at a time.

## The first trap: an isolated profile is not an isolated prompt

OMP provides a native profile switch:

```powershell
omp --profile zero
```

A profile isolates settings, authentication, sessions, and caches. That sounds sufficient, but it does not necessarily isolate instruction discovery. A global instruction file under the real user home can still enter the rendered prompt.

Similarly:

```powershell
omp --no-rules
```

suppresses rule loading, but a separately implemented context-file provider may still discover a global `AGENTS.md`.

The test is not whether the command line *looks* clean. The test is whether the rendered prompt is clean.

For a true baseline, I combine three boundaries:

1. a dedicated OMP state directory;
2. a clean process-scoped `HOME` and `USERPROFILE`;
3. an empty working directory outside the normal project tree.

The environment changes exist only for the child process and are restored when OMP exits.

```powershell
$realHome = $env:HOME
$realUserProfile = $env:USERPROFILE
$realLocation = Get-Location

try {
    $env:HOME = "C:\agent-zero"
    $env:USERPROFILE = "C:\agent-zero"
    $env:PI_CODING_AGENT_DIR = `
        "$realUserProfile\.omp\profiles\zero\agent"

    Set-Location "C:\agent-zero"
    omp --no-extensions --no-skills --no-rules
}
finally {
    $env:HOME = $realHome
    $env:USERPROFILE = $realUserProfile
    Set-Location $realLocation
}
```

This is less elegant than one profile flag, but it is an honest boundary. Configuration isolation and prompt isolation are different concerns.

## Measure the harness, not the model's recollection

OMP's RPC mode exposes the rendered state directly:

```bash
printf '{"id":"1","type":"get_state"}\n' \
  | omp --mode rpc --no-session
```

The useful fields are:

- `contextUsage.tokens`: static tokens loaded before the conversation;
- `systemPrompt`: rendered instruction blocks;
- `dumpTools`: tool schemas included in the prompt.

This produced the following progression on a 1.1-million-token model context window:

| Configuration | Tools | Static tokens | Approx. window |
|---|---:|---:|---:|
| Bare OMP defaults | 11 | 14,274 | 1.3% |
| Repository core | 6 | 6,219 | 0.6% |
| Repository core + web search | 7 | 6,517 | 0.6% |
| No tools | 0 | 2,676 | 0.25% |

The no-tool configuration is the absolute calibration floor, but it cannot do useful work. The seven-tool version is the practical baseline.

## Tool schemas are part of the prompt

The largest reduction did not come from deleting prose. It came from selecting tools.

OMP's bare default exposed:

```text
read, bash, edit, eval, glob, grep,
task, hub, todo, web_search, write
```

Agent Zero retains:

```text
read, bash, edit, write, grep, glob, web_search
```

`read` also fetches URLs, so web retrieval needs no second URL-specific tool. `web_search` remains because an agent that cannot discover unknown sources is not a useful engineering baseline.

The excluded tools are deliberate:

| Tool | Why Zero excludes it |
|---|---|
| `eval` | Persistent language kernels add substantial schema and runtime capability. |
| `task` | Subagents introduce delegation policy and another execution layer. |
| `hub` | Agent/process coordination belongs to multi-agent operation, not the baseline. |
| `todo` | Task-state management is workflow policy rather than core repository access. |

Adding `web_search` cost only 298 static tokens. That is an excellent trade: measurable research capability for roughly 0.03% of the model window.

## What existing minimal agents taught me

There is already a well-known project named [Agent Zero](https://github.com/agent0ai/agent-zero). It is a capable autonomous framework with memory, tools, and subordinate agents—the name overlaps, but the design goal is almost the opposite.

Closer prior art includes:

- [Kon](https://github.com/0xku/kon), a deliberately minimal and opinionated coding agent;
- [mini-coding-agent](https://github.com/rasbt/mini-coding-agent), a readable implementation of the essential agent loop.

Those projects minimize the harness itself. This experiment keeps OMP constant and minimizes what OMP loads. That distinction matters: Agent Zero is not a competitor to a coding harness; it is a baseline *inside* one.

## Use Zero as a differential measurement

The useful workflow is simple:

1. Measure Agent Zero.
2. Enable one skill, extension, tool, or instruction file.
3. Measure again.
4. Keep the addition only if its recurring value justifies its recurring context cost.

Record each result as a delta from the immediately preceding measurement:

```text
marginal cost = tokens after enabling capability
              - tokens before enabling capability
```

System prompts and tool schemas evolve between harness versions, so compare the control and experiment using the same OMP revision. A delta measured across two versions conflates the capability with harness changes.

## The broader pattern

Specialized agents are useful because persistent context gives them continuity. Persistent context is also a tax paid on every turn.

Without a control group, that tax becomes invisible. Every instruction sounds individually reasonable; together they become a second codebase sitting in the model's input.

Agent Zero makes the trade explicit:

- **Start with capability, not ceremony.** Files, shell, edits, search, and URL retrieval are enough for a useful baseline.
- **Treat tool schemas as context.** Removing unused tools can save more than editing prompt prose.
- **Separate state isolation from prompt isolation.** A clean profile may still inherit global instructions.
- **Measure marginal cost.** Total size is interesting; the delta caused by one addition is actionable.
- **Keep the absolute floor as a calibration point.** A 2,676-token no-tool launch tells us what the harness itself costs, even though it is not a working agent.

The objective is not the smallest possible prompt. It is the smallest prompt that still performs the job—and evidence for every token added beyond it.

## Update: plain `omp` is not yet the control group

I later tried to make the working directory self-activating: enter the Agent Zero directory, run `omp`, and get the same baseline without remembering a launcher command.

OMP can do part of this natively. A project `.env` can select the isolated state directory:

```dotenv
PI_CODING_AGENT_DIR=C:/path/to/.omp/profiles/zero/agent
```

Project settings can also remove configured extensions and disable skills:

```yaml
# .omp/config.yml
extensions: []
skills:
  enabled: false
```

That makes plain `omp` use the correct authentication, model cache, sessions, and project settings. It does **not** make it equivalent to the control-group launcher.

Measured on the same one-million-token model context:

| Entry point | Tools | Static tokens | Approx. window |
|---|---:|---:|---:|
| Plain `omp` with project `.env` and config | 11 | 15,781 | 1.6% |
| Agent Zero launcher | 7 | 5,630 | 0.6% |

The remaining delta comes from controls that OMP 17.2.15 exposes only as command-line flags:

```text
--no-rules
--tools read,bash,edit,write,grep,glob,web_search
```

This is an important distinction when benchmarking agents: **selecting the right profile proves state isolation; it does not prove prompt equivalence**. Compare rendered state, tool count, and static tokens—not directory names or configuration intent.

I filed [OMP issue #8346](https://github.com/can1357/oh-my-pi/issues/8346) to request project/profile settings with semantic parity for rule loading, extension discovery, and tool allowlisting. Until those settings exist, the dedicated launcher remains the reproducible control-group entry point.

## Source

- [Oh My Pi](https://github.com/can1357/oh-my-pi) — coding-agent harness and RPC state interface used for the measurement.
- [Kon](https://github.com/0xku/kon) — minimal, opinionated coding-agent prior art.
- [mini-coding-agent](https://github.com/rasbt/mini-coding-agent) — readable minimal agent-loop implementation.
- [Agent Zero](https://github.com/agent0ai/agent-zero) — established autonomous-agent framework with the same name but a different design objective.
