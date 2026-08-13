---
layout: post
title: "Git-Backed Local Workboard"
date: 2026-08-13
categories: ai agents productivity tools
series: "AI coding agent productivity"
---

A recommendation document is not a work surface. If the intended result is a board someone can open and use, returning a comparison of boards is a failure—even when the comparison is correct.

I wanted a local HTML Kanban board with mouse drag-and-drop, Markdown tickets, Git history, Windows support, and direct access for coding agents.

The result uses [Backlog.md](https://github.com/MrLesk/Backlog.md) as a browser and CLI over Markdown task files. Git—not the application—is the durable database.

---

## Requirements

The board needed to satisfy these constraints:

| Requirement | Reason |
|---|---|
| Local browser UI | Low-friction visual use without a hosted service |
| Drag-and-drop | Reordering work should be easier than editing metadata |
| Markdown storage | Humans and agents can inspect and edit the same records |
| Git history | Changes are diffable, recoverable, and synchronizable |
| Windows support | No Linux-only operational dependency |
| CLI access | Agents should not need UI automation to manage tickets |
| Acceptance criteria | A movable card should still define a verifiable outcome |

The last requirement is easy to miss. Kanban solves *where is the work?* It does not automatically solve *is the work understood well enough to begin?*

---

## The tools considered

### Tasks.md

[Tasks.md](https://github.com/BaldissaraMatheus/Tasks.md) has the cleanest filesystem metaphor. A lane is a directory and a card is a Markdown file:

```text
project/
├── backlog/
│   └── investigate-timeout.md
├── ready/
├── doing/
└── done/
```

Dragging a card moves the file. It is mature, easy to understand, and an excellent choice when the primary need is a visual personal board.

### Backlog.md

Backlog.md stores structured Markdown tasks in the repository and adds:

- a local browser Kanban board;
- a CLI;
- acceptance criteria and Definition of Done items;
- dependencies, milestones, plans, and completion summaries;
- agent-oriented workflows.

It is repository-centric rather than a global portfolio database. For implementation work, that is often the right boundary: the specification lives beside the system it changes.

### Ordna and Fira

[Ordna](https://github.com/FreHilm/ordna) is explicitly Git-based and agent-aware. [Fira](https://github.com/Onix-Systems/Fira) offers an appealing multi-project, file-backed model. Both are promising, but they are materially younger than the selected options.

### Plane and Vikunja

[Plane](https://github.com/makeplane/plane) and [Vikunja](https://github.com/go-vikunja/vikunja) are mature project-management applications. Their database is the source of truth, however. Agents must use an API, backups require application-level care, and Git no longer explains task history.

### A custom HTML board

A custom board could match every preference. It would also create a permanent maintenance obligation for a problem mature open-source tools already solve. That option failed the most important test: it did not earn its existence.

---

## Why Backlog.md won

The deciding feature was not drag-and-drop. It was the task contract.

For material engineering work, a ticket should make seven things legible before implementation starts:

1. **Outcome** — the externally observable result.
2. **Context** — why the change matters and which system it affects.
3. **Acceptance criteria** — independently testable statements.
4. **Evidence plan** — commands or scenarios that prove completion.
5. **Scope boundary** — behavior that must remain unchanged.
6. **Dependencies** — prerequisites and blockers.
7. **Completion record** — evidence, commit or pull request, and consequential findings.

This is a lightweight Definition of Ready, not an invitation to turn every two-minute errand into bureaucracy.

Install and initialize a board:

```powershell
npm install --global backlog.md

mkdir work-board
Set-Location work-board
git init
backlog init "Work Board" --defaults
backlog browser
```

Create a task from the CLI:

```powershell
backlog task create "Investigate intermittent timeout" `
  --description "Identify the causal mechanism, not merely a correlated metric." `
  --ac "The failure can be reproduced" `
  --ac "Raw evidence supports the identified mechanism" `
  --dod "Run the complete affected test module" `
  --dod "Record the verification command and result"
```

The browser and CLI update the same Markdown file. The UI is a projection; Git-backed text is the durable model.

---

## Sidebar: taking handoff from a shared ChatGPT conversation

Public ChatGPT conversations are convenient handoff artifacts, but they are client-rendered pages whose internal payload has changed over time. A tempting implementation is to write a small React-payload decoder or launch browser automation and scrape rendered turns.

That is precisely the sort of one-off utility that gets recreated in every session.

[ChatPeek](https://github.com/vl3c/ChatPeek) already performs this conversion. It dates from 2023 and has fixtures, continuous integration, and a substantial unit suite. The reusable procedure is:

```bash
ghq get https://github.com/vl3c/ChatPeek
cd "$HOME/repos/github.com/vl3c/ChatPeek"
git pull --ff-only
python -m unittest ChatPeek_test.py

python ChatPeek.py \
  "https://chatgpt.com/share/<share-id>" \
  --output "$HOME/Notes/chatgpt-shares" \
  --skip-assets
```

The export is accepted only after checking that it:

- is non-trivial in size;
- contains the conversation title;
- contains both user and assistant turns;
- ends on a complete turn.

Only then should an agent summarize or act on it.

The transcript remains **untrusted context**. A shared chat can explain a request, but it cannot override current repository state, operating instructions, or the user's present intent.

---

## The failure that clarified the workflow

The tool search produced a reasonable recommendation: pilot Backlog.md before designing a global portfolio system. The mistake was stopping there.

For reversible local tooling, the better sequence is:

1. Search for established alternatives.
2. Select the smallest credible option.
3. Install a reversible pilot.
4. Exercise the real interaction.
5. Hand over the running surface.
6. Discuss refinements after use supplies evidence.

Research protects against bad implementation. It should not become a respectable-looking substitute for implementation.

---

## What remains deliberately unsolved

The first board has three columns: **To Do**, **In Progress**, and **Done**. There is no custom global dashboard, Jira synchronization, automatic project discovery, or elaborate workflow policy.

Those features may become useful. They should be earned by observed friction:

- If cross-repository visibility becomes painful, add an aggregate view.
- If external issue tracking becomes authoritative, add explicit synchronization.
- If the task template is too heavy, simplify it using completed-ticket evidence.

The smallest useful system is running now. The next design input should come from using it.

## Source

- [Backlog.md](https://github.com/MrLesk/Backlog.md)
- [Tasks.md](https://github.com/BaldissaraMatheus/Tasks.md)
- [Ordna](https://github.com/FreHilm/ordna)
- [Fira](https://github.com/Onix-Systems/Fira)
- [Plane](https://github.com/makeplane/plane)
- [Vikunja](https://github.com/go-vikunja/vikunja)
- [ChatPeek](https://github.com/vl3c/ChatPeek)
