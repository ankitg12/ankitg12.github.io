---
layout: post
title: "goalsq: Daily Priorities in a Logseq Journal"
date: 2026-09-25
categories: tools productivity logseq
series: "Personal Productivity"
---

I did not have a daily tracking system I kept using. I had tried other ways to track what needed doing, but each one meant returning to another place to maintain a list. Meanwhile, I was already adding notes to my Logseq journal from the terminal with [lsq](https://github.com/jrswab/lsq).

The journal was the place I actually visited. So why keep today's priorities somewhere else?

That question led to [goalsq](https://github.com/ankitg12/goalsq). It keeps the day's list inside the dated Logseq Markdown file, in a `[[Goals]]` block:

```markdown
- [[Goals]]
	- NOW Draft the review
		- Outline saved
	- LATER Check the test results
- 15:00 Notes from the meeting
```

I can edit those lines in Logseq. At the terminal, I can add an item, mark what matters now, attach a note, or mark it done:

```sh
goalsq add -s now "Draft the review"
goalsq add "Check the test results"
goalsq -s now
goalsq 1 set note "Outline saved"
goalsq done 1 "Review sent"
```

There is no second database to reconcile. The command reads and writes the journal I already use. It shows `NOW`, `LATER`, and `DONE` separately and keeps the journal's item numbers when I filter the display. Completed items move below open ones when I write to the file.

Despite the name, I use it for **daily priorities**, not to measure progress toward a lifelong goal. Some entries are outcomes for the day; others are actions. The useful habit is choosing what deserves attention today, in a place I will return to tomorrow.

`lsq` remains my fast path for journal notes. `goalsq` handles one block in the same Markdown journal; it does not call `lsq` or need the Logseq desktop API.

If your graph lives outside `~/Logseq/journals`, point `GOALSQ_JOURNALS_DIR` at its `journals` directory. The package needs Python 3.10 or newer and can be installed from GitHub with [uv](https://docs.astral.sh/uv/concepts/tools/):

```sh
uv tool install --python 3.12 git+https://github.com/ankitg12/goalsq.git
```

## Source

- [goalsq — code, tests, and setup](https://github.com/ankitg12/goalsq)
- [lsq — journal capture CLI](https://github.com/jrswab/lsq)
- [uv — installing CLI tools](https://docs.astral.sh/uv/concepts/tools/)
