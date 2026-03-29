---
layout: post
title: "Patterns of Agentic AI"
date: 2026-03-29
categories: ai agents
---

Agentic AI tools — Claude Code, Codex, Gemini CLI, Cursor — are not just assistants. They read files, run commands, write code, and operate across entire sessions with growing autonomy. How you configure them at session start has an outsized impact on what they do and how fast they do it.

This post documents a pattern library I built for structuring agent sessions: [agent-commons](https://github.com/ankitg12/agent-commons). It covers three patterns across a spectrum from zero configuration to full shared memory.

---

## The spectrum

```
agent-zero  →  agent-one  →  agents-shared
 no memory      rules only    shared memory + identity
```

Each pattern is a drop-in template. The right choice depends on the task.

---

## agent-zero — pure model defaults

No instruction files. No memory. No session protocol. The agent starts clean, with nothing loaded beyond what the tool does by default.

This sounds like the absence of a pattern. It is — intentionally.

Most agent configuration advice goes in one direction: add more. More rules, more context, more memory. agent-zero goes the other way. A model with no framing is useful for:

- **Fast one-off tasks** — tasks that don't need any prior context and where loading memory is pure overhead
- **Exploratory sessions** — you want the model's unguided judgment, not constrained by rules you wrote last week
- **Debugging agent behaviour** — when a configured agent does something unexpected, running agent-zero on the same task reveals whether the issue is the model or your configuration

The implementation challenge is that most agentic tools load global configuration automatically. Claude Code reads `~/.claude/CLAUDE.md` on every startup regardless of where you launch it. To truly start clean, you need to redirect the home directory to one that contains only auth — no rules files, no hooks, no MCP servers.

On Windows:

```powershell
$env:USERPROFILE = "C:\path\to\clean-home"
$env:HOME        = "C:\path\to\clean-home"
claude
```

The clean home needs only enough to authenticate — on an enterprise proxy setup, that's a minimal `.claude.json` with `skipApiKeyConfirmation` and `primaryApiKey`. The proxy credentials come from environment variables set system-wide, so they're inherited regardless of home directory.

agent-commons includes a `home-template/` and launcher scripts (`launch.ps1`, `launch.sh`) that handle this.

---

## agent-one — one file, consistent behaviour

One instruction file (`AGENTS.md` / `CLAUDE.md` / `GEMINI.md` / `.cursorrules`). No memory, no session protocol, no identity loading. Just behavioural rules:

```markdown
## Rules
- Be terse. No filler, no preamble.
- Do the task. Don't ask unnecessary questions.
- When uncertain about something destructive, state what you'd do before doing it.
- If stuck after two attempts, stop and ask.
```

This is the pattern you want when you run the same kind of work repeatedly and want consistent agent behaviour without the overhead of a memory system. The agent won't remember previous sessions, but it will behave the same way in each one.

`CLAUDE.md`, `GEMINI.md`, and `.cursorrules` are symlinks to `AGENTS.md` — one file to edit, all harnesses pick it up.

---

## agents-shared — multi-agent shared memory

The most structured pattern. Multiple agents (different tools, different models) work on the same project and share:

- **Identity** — a common context file (`IDENTITY-common.md`) read by every agent
- **Knowledge** — mutable state files in `knowledge/` (cluster state, deployed versions, active configuration)
- **Narrative** — an append-only session log in `notes/YYYY-MM-DD.md` that accumulates across sessions

The folder is self-contained. Copy `agents-shared/` into your project, add one subdirectory per tool, and each agent instance shares the same layer above it:

```
agents-shared/
├── IDENTITY-common.md   ← read by every agent
├── KNOWLEDGE.md
├── knowledge/
├── notes/
├── claude/              ← Claude Code instance
│   ├── AGENTS.md
│   └── IDENTITY.md      ← Claude-specific notes
└── codex/               ← Codex instance
    ├── AGENTS.md
    └── IDENTITY.md
```

**Two memory layers, kept separate:**

| Layer | Location | Rule |
|-------|----------|------|
| State | `knowledge/*.md` | Mutable. Update when reality changes. |
| Narrative | `notes/YYYY-MM-DD.md` | Append-only. Never edit past days. |

The separation matters. State files are ground truth — they get updated when facts change. Notes are history — they record what happened, what was tried, what was decided. Mixing the two means either stale state or lost history.

**Session protocol** (built into `AGENTS.md`):

Start: read common identity → read knowledge index → read today's and yesterday's notes → verify state before acting.

End: append summary to today's notes → update any knowledge files that changed → commit and push.

---

## Why not always use agents-shared?

Because loading context has a cost — in tokens and in latency. A session that starts by reading 5 files before doing anything is slower to get going and burns context on content that may not be relevant to the task.

The pattern should match the task:

| Task | Pattern |
|------|---------|
| Debug a one-off issue | agent-zero |
| Repeated task needing consistent behaviour | agent-one |
| Long-running project with multiple agents | agents-shared |

---

## On symlinks

All harness files (`CLAUDE.md`, `GEMINI.md`, `.cursorrules`) in agent-one and agents-shared are symlinks to `AGENTS.md`. Edit one file; all tools pick it up. Git tracks symlinks as `mode 120000`.

One caveat: Windows users need `core.symlinks=true` in git config (enabled by default when git for Windows is installed with admin rights). Without it, symlinks become text files containing the target path.

---

## The repo

[github.com/ankitg12/agent-commons](https://github.com/ankitg12/agent-commons) — fork or clone, pick a pattern, drop it in your project.

Built on the [agent-kernel](https://github.com/oguzbilgic/agent-kernel) idea by [@oguzbilgic](https://github.com/oguzbilgic).
