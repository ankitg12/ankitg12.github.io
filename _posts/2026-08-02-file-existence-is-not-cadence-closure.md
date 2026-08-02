---
layout: post
title: "Your Review File Exists. The Review Still Didn't Happen."
date: 2026-08-02
categories: ai agents productivity engineering
series: "AI coding agent productivity"
---

A weekly review file existed exactly where the automation expected it. The week still had not been reviewed.

The file was created on Monday with a plan. By Sunday it contained intentions, outcomes, and evidence—but no retrospective. A startup check based on filename existence reported nothing wrong.

That is the trap: **existence is not completion**.

This matters whenever one artifact is filled progressively. A file created at the start of a period can prove that the period was opened, but it cannot also prove that the period was closed.

## The naive check

The first design looked reasonable:

|Cadence|Expected artifact|When to nudge|
|---|---|---|
|Weekly|`weekly/YYYY-Www.md`|Current ISO-week file is missing|
|Monthly|`monthly/YYYY-MM.md`|Previous month's file is missing|
|Quarterly|`quarterly/YYYY-QN.md`|Previous quarter's file is missing|

The monthly and quarterly rules work because their rollups are written after those periods end. A missing file means a missing close.

Weekly is different. One file holds two state transitions:

```markdown
## Plan: Week of ...

...written when the week opens...

## Retrospective: Week of ...

...appended after the week closes...
```

Checking only `weekly/2026-W31.md` answers one question:

> Was week 31 opened?

It does not answer:

> Was week 31 closed?

The false negative appeared at the boundary. On the last day of W31, its file existed, so the hook stayed quiet. On Monday, W32 could be opened while W31 remained unreviewed. The system had confused a container with a completed process.

## Model transitions, not files

The corrected weekly rule has two independent checks:

1. **Opening:** the current Monday-first ISO week has a file.
2. **Closure:** the previous ISO week's file contains a retrospective heading.

They can both fire on the same session start:

```text
No weekly review for 2026-W32 — run /weekly-review to close it out.
No weekly retrospective for 2026-W31 — run /weekly-review to close it out.
```

That is not duplicate noise. The lines describe two different missing transitions: open this week, close last week.

The complete cadence model became:

|Check|Period examined|Completion evidence|
|---|---|---|
|Daily close|Previous calendar day|Journal contains `## Daily Close`|
|Weekly open|Current Monday-first ISO week|Weekly file exists|
|Weekly close|Previous ISO week|Weekly file contains a retrospective heading|
|Monthly close|Previous calendar month|Monthly rollup exists|
|Quarterly close|Previous quarter|Quarterly rollup exists|

The asymmetry is deliberate. You cannot demand that the current day, week, month, or quarter be closed while it is still in progress.

## The date math is small, but boundary-sensitive

JavaScript's default week assumptions are easy to get wrong, especially around Sundays and year boundaries. The hook converts the local calendar date to a UTC date used only for ISO-week arithmetic:

```ts
function isoWeek(today: Date): { year: number; week: number } {
  const date = new Date(Date.UTC(
    today.getFullYear(),
    today.getMonth(),
    today.getDate(),
  ));

  const day = date.getUTCDay() || 7; // Monday=1, Sunday=7
  date.setUTCDate(date.getUTCDate() + 4 - day);

  const year = date.getUTCFullYear();
  const yearStart = new Date(Date.UTC(year, 0, 1));
  const week = Math.ceil(
    ((date.getTime() - yearStart.getTime()) / 86_400_000 + 1) / 7,
  );

  return { year, week };
}
```

Moving to Thursday determines the ISO week-year correctly. That detail matters around New Year's Day, where the calendar year and ISO week-year can differ.

The previous weekly period is not "week number minus one." It is the ISO week containing the date seven days earlier:

```ts
const previousWeekDate = new Date(
  today.getFullYear(),
  today.getMonth(),
  today.getDate() - 7,
  12,
);
const previousWeek = isoWeek(previousWeekDate);
```

Subtracting seven calendar days lets the date implementation handle month and year rollover.

## Match a semantic marker, not the whole heading

The close check needs a stable marker, but exact heading text is too brittle. A retrospective may legitimately be titled:

```markdown
## Retrospective: Week of Jul 20–26
```

The implementation therefore uses a case-insensitive prefix match:

```ts
/^##\s+Retrospective\b/im
```

This tolerates capitalization and descriptive suffixes without accepting unrelated sections such as `## End-of-week evidence`.

There is a useful boundary here: automation should tolerate harmless formatting variation, but it should not silently redefine the convention. If the team wants another heading to mean "closed," that is a process decision, not a regex improvisation.

## Startup nudges should fail open

This check runs during an agent's `session_start` lifecycle. A broken productivity reminder must never prevent the session it is supposed to help.

The hook follows four rules:

1. Read local files only; no service dependency.
2. Catch unreadable-directory and unreadable-file errors.
3. Emit nothing when all checks pass.
4. Display missing checks without triggering an agent turn.

In an [Oh My Pi extension](https://github.com/can1357/oh-my-pi/blob/main/docs/extensions.md), the final delivery is intentionally small:

```ts
pi.on("session_start", async () => {
  const nudges = await getCadenceNudges();
  if (nudges.length === 0) return;

  pi.sendMessage({
    customType: "cadence-nudge",
    content: nudges.join("\n"),
    display: true,
  }, { triggerTurn: false });
});
```

This is discoverable, not enforced. The person can ignore the nudge. The system does not open an interactive wizard, block startup, or print an "all clear" message every session.

A reminder that constantly announces its own health becomes another maintenance burden.

## Verify boundaries, not just helpers

The smallest useful verification matrix covered three states:

|Scenario|Expected output|
|---|---|
|Sunday, current weekly file exists, previous week has retrospective|No weekly nudge|
|Monday, new weekly file missing, previous week lacks retrospective|Two weekly nudges|
|All daily/weekly/monthly/quarterly evidence exists|Zero output bytes|

The Monday case is the regression test. It fails if the implementation collapses weekly opening and closure back into one existence check.

The no-op case matters equally. Non-blocking automation should prove that success is silent, not merely that failure is loud.

## The general pattern

This bug is not specific to review notes. It appears anywhere a durable object is created before its workflow completes:

- a pull request exists but has not been reviewed;
- an incident document exists but has no resolution section;
- a migration file exists but has not been applied;
- a backup file exists but has not been restored in a test;
- a project page exists but has no decision or owner.

The reliable question is not:

> Does the artifact exist?

It is:

> What observable transition proves that this stage completed?

Sometimes the answer is another file. Sometimes it is a heading, status field, timestamp, database constraint, or successful restore. Whatever the marker is, check the marker that represents completion—not the container that happens to hold it.

## Source

- [Oh My Pi](https://github.com/can1357/oh-my-pi)
- [Oh My Pi extension documentation](https://github.com/can1357/oh-my-pi/blob/main/docs/extensions.md)
- [ISO 8601:2004 — Data elements and interchange formats](https://cdn.standards.iteh.ai/samples/40874/e4537809b46a40c3b1711fcb197559af/ISO-8601-2004.pdf)
- [Node.js file-system promises API](https://nodejs.org/api/fs.html#promises-api)
