---
layout: post
title: "Every friction line is a missing fact: measuring where an LLM test agent gets lost"
date: 2026-10-29
categories: ai agents productivity
series: "AI coding agent productivity"
---

An LLM agent that runs a hardware test case took 34 minutes for a test whose hardware time is about 6 minutes.

Every run that finished passed, and the verdict was right. The time went to the agent finding its way: searching for scripts, running `--help`, re-reading files, and writing its own copies of commands that already existed. A pass/fail metric does not show any of this.

This post describes the two checks I now run after every agent run, what they found, and the fixes. Between the first and the latest complete run, the same test went from 34.8 to 22.6 minutes, with no change to the test.

## The setup

- A test-automation agent framework drives a test case written in Markdown: numbered steps, each with a committed `bash` fence.
- A main agent orchestrates; a subagent runs the steps on two hosts; an analyst subagent writes the report.
- The agents run in a coding-agent harness and write a JSONL transcript: every tool call, every tool result, every thinking block, with timestamps.

The transcript is the raw material. Everything below is a script over it.

## Check 1: rule classes over tool calls

A small script walks the transcript and puts each tool call that is not progress into a class:

|Class|Rule|What it means|
|---|---|---|
|search|`glob`, `find`, recursive `grep`/`ls`|The agent did not know where something is|
|help|a command with `--help`|The docs did not give the full command|
|reread|the same file read a third time|The agent lost a fact between turns|
|write|a file the agent authored (write tool, `cat > file`, heredoc)|The agent copied or invented content|
|preamble|a shell call that re-declares 3+ variables from an earlier call|No environment carries between calls|
|error|a tool call with a non-zero exit|A documented command was wrong or misused|
|confused|thinking text with "where is", "can't find", "bypass", "workaround"|The agent says it is lost|

The rule I follow: **every line is a fact the documents did not give, or a command the agent got wrong. Fix it before the next run.** Do not tune the prompt; fix the source of truth the agent reads.

## Check 2: a cheap model judges every thinking block

Rules miss friction that the agent never acts on in an obvious way. So the script also prints every thinking block as JSONL, and a small model labels each one:

- `on-track`: follows a documented step or reports a result
- `missing-fact`: unsure about a path, host, variable or file the docs should have given
- `workaround`: plans to bypass, rewrite or hand-copy a documented step
- `guessing`: assumes something without evidence

On one run: 50 thinking blocks, 9 seconds, a quarter of a cent. 42 were on-track. Of the 8 others, 4 were not visible to the rule classes, for example:

- the subagent could not tell how a step number maps to the staging file the framework writes;
- a variable set in one shell call was gone in the next, so a helper script was not found;
- the subagent could not tell whether clients on two hosts may start at the same time.

A label is a pointer, not proof. I read each flagged block before I fix anything.

## What the checks found

Three root causes explain most of the lost time.

**1. An empty environment variable in the harness.** The framework's docs call scripts as `${PLUGIN_ROOT}/scripts/x.py`. The harness did not set that variable. The agent then ran `env`, `--help` on several scripts, and a `find /` to locate them. One `export` in the launcher removed all of it.

**2. A command shape the shell tool mangles.** The close-out step piped a heredoc into `python3 "${ROOT}/scripts/write_report.py"`. With a heredoc in the same call, the harness's shell tool split `"${ROOT}/x"` into wrong arguments. I reproduced it in isolation:

```bash
export CPR=/tmp/cpr
python3 -c 'import sys;print(sys.argv[1:])' "${CPR}/x.py"     # fine
cat << 'EOF' | python3 -c 'import sys;print(sys.argv[1:])' "${CPR}/x.py"
hi
EOF
# the second call does not receive /tmp/cpr/x.py
```

The agent saw the error, decided the behavior was "peculiar", and wrote a `python3 -c` wrapper around the documented script. The report was still written, so the run passed. Only the friction check showed it.

**3. A step the tool could not run verbatim.** Test steps run with `run --tc-file X --step N`: the tool extracts the committed fence and runs it as written. The suite setup lived under a different heading, so the extractor found 0 steps, and the agent copied 59 lines into `/tmp/suite_setup.sh` by hand. That copy matched the source except for indentation, but a run that executes an agent's copy is not running the committed test. The fix is a `--section "Suite Setup"` option on the same extractor.

## Results

|Run|Fixes before it|Prompt to verdict|
|---|---|---|
|A|earlier work, before the friction checks existed|34.8 min|
|B|the same as A|passed; time not usable (the laptop suspended mid-run)|
|C|exact commands in the subagent's memory file|stopped early (a provider content-filter stop)|
|D|harness exports the plugin root; "run documented commands as written"|22.6 min|

Run D still had friction: 26 calls re-declared the same variables, the hand-copied setup, and the close-out workaround. Those are the fixes for the next run.

## What this says about where the LLM belongs

Each fix removes a decision from the agent. Taken to the end, that turns the agent into a slow script runner. About 6 of the 34 minutes were hardware time.

My conclusion so far: for a test that already passes, the steps should run as code, and the LLM is worth its cost in three places: writing the test case from prose, triaging a failure, and explaining the verdict. That is how CI uses people: they look at red builds. I have not built that mode yet.

## Not checked

- Whether the heredoc mangling comes from the shell tool or from a rule layer on top of it.
- How stable the judge labels are across models or repeated runs.
- The time saved by each single fix; the runs changed several things at once.

## Source

- Step-level faults that outcome-only judges miss: [trajectory-judge, arXiv 2609.00038](https://arxiv.org/html/2609.00038v2)
- Step scoring and error spread in agent trajectories: [AgentEval, arXiv 2604.23581](https://arxiv.org/html/2604.23581)
