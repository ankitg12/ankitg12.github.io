---
layout: post
title: "Who Runs the Test When the Model Is Down?"
date: 2026-10-23
categories: ai agents testing
series: "AI coding agent productivity"
---



A team I know had the fashionable argument: let the agent run every test case
straight from its markdown file — the reasoning models are smart, they handle
the edge cases. I took the unfashionable side: agents should write code, and
code should run the suite. Four engineering teams, four sets of numbers, all
published this year, settled it for me. None of them runs per-case agents as
the suite. Every one of them lets agents produce something deterministic — and
then runs that something free.

---

## 1. Slack priced the run — $15–30 a pop (Jun 2026)

[Slack's engineering team ran 200+ agentic E2E workflows](https://slack.engineering/agentic-testing-where-agents-fit-in-the-e2e-testing-stack/)
across three execution models: agent-through-MCP, agent-through-CLI, and
agent-generates-Playwright-tests. The metered runs cost **$15–30 per
execution** and took 5–11 minutes. The generated tests executed in 32–45
seconds — generation a one-time cost, amortised to nothing over a quarter of
CI runs.

Their sleeper finding: 3.5–7M tokens per flow, driven by **turn count (40–85
turns of re-sent context)**, not model size. A smaller model burned more than
the flagship. Most of the bill was retransmission of previously seen content.
And generated tests held ~8% failure on simple flows but fell to **~48% on
complex ones** — dying at 70–80% progress on a final interaction.

Slack's conclusion is the frame for the other three: deterministic tests as
the CI foundation, agentic runs as the exploratory layer — debugging flaky
workflows, reproducing production bugs, probing complex behaviour. Placement,
not replacement.

---

## 2. Atlassian made the PR the unit of work — 70%→82% (Oct 2026)

[One Atlassian commerce team handed coverage to a scheduled agent](https://www.atlassian.com/blog/development/using-agentic-pipelines-to-turn-test-coverage-into-daily-habit):
find one thinly tested file, write the missing tests, open one small pull
request. Reviewers merged ~34 agent-opened PRs in a month; repository coverage
went 70%→82%.

Their line: *the pull request was the useful unit of work.* Agent authors,
human merges. When reviewers flagged the same weak pattern twice, the team
fixed the prompt once instead of leaving the same comment forever. The prompt
became part of the test system — conventions, mocking rules, off-limits
production code, all versioned beside the code it governs.

Note what the agent never does: execute the suite per case. It produces the
artifact; the pipeline runs it free.

---

## 3. OpenObserve built the council — 380→700+, flake down 85% (Jan 2026)

[OpenObserve's "Council of Sub Agents"](https://openobserve.ai/blog/autonomous-qa-testing-ai-agents-claude-code/)
runs eight specialised roles — Analyst, Architect, Engineer, Sentinel (a hard
quality gate that blocks the pipeline), Healer (iterates to green, up to 5
tries), Scribe — over Playwright. Six months in: feature analysis 60 min→5
min, coverage 380→700+ tests, flaky tests down 85%.

They came to write tests and accidentally became better debuggers. And their
architecture lesson mirrors the field: one super-agent failed; bounded
specialists won.

The caveat: the Healer runs tests agentically, up to five iterations each —
metered inference at qualification time. Their output is Camp 2; their
machinery still leans Camp 1. The outage test is the question their roadmap
has not yet answered.

---

## 4. Kong sent agents after the flakes (Jun 2026)

[Kong aimed agentic AI at its flakiest gateway tests](https://konghq.com/blog/engineering/how-we-used-agentic-ai-to-fix-kong-gateways-flakiest-tests):
identify, fix, verify. Agents as repair crew on deterministic suites — not
executors of markdown cases. The suite stays deterministic; the agent keeps it
that way.

---

## What the four agree on

- **Agents author; determinism executes.** Every shop converges here. The
  metered run is the exploratory tool, not the regression suite.
- **The artifact is the interface.** PR (Atlassian), audited Playwright code
  (OpenObserve), fixed specs (Kong), generated tests (Slack). Humans review
  artifacts; pipelines run them.
- **Specialists beat generalists.** One super-agent failed at OpenObserve;
  Slack's MCP beat CLI on infrastructure, not reasoning; Atlassian's one-file
  scope beat broad sweeps.
- **Tighter inputs beat bigger models.** Structured YAML over loose prose,
  verified selectors only, repo conventions in the prompt.

[ accompany with the TestMu gate rule](https://www.testmuai.com/blog/agentic-test-run-cost-vs-ci/):
agent checks on the flows whose failure would hurt most; replacing a whole
regression suite on day one buys risk without buying information.

---

## The outage test

Models go down. Lab windows do not wait. A suite that needs inference to
execute cannot qualify a release on a bad-model day. Scripts run. The scarce
resource is the lab window, not the model.

Two shorter objectives also beat one long one: they cost less *and* fail more
usefully, because you learn which half broke.

---

## Dignity

Authorship and judgment — designing the flow worth checking, reading the
failure that matters, owning the boundary the test guards — is engineering.
Clicking re-run across six live agent contexts is operations. As [Kyle Lee put
it after running a fleet across three machines](https://ai.engineer/talks/i-run-a-fleet-of-ai-agents-across-three-machines-here-s-what-broke):
*the first thing that broke wasn't a machine — it was me.*

Keep the engineer as author. Let agents write, let CI run free, let agents
repair. Both camps survive — each where it wins.

---

## Update (2026-10-23): not fashionable — metered while nobody watches

A reader pushed back on the lede: is the run-everything-through-agents side
really "fashionable," or is it something else? Fair challenge. The evidence
says it is a split with a clock on it.

One developer's [$37,901.73 Bedrock bill](https://hemantvirmani.substack.com/p/someone-just-got-a-38000-aws-bill)
came not from a leak or a loop but from prompt caching silently failing
across a proxy chain — 6.47 billion uncached input tokens. The line that
matters: *supported is not the same as actually using correctly.* At
enterprise scale, a 2,000-engineer org runs [$1.66M a month across five coding
tools with no common chargeback table](https://futureagi.com/blog/enterprise-llm-gateway-cost-tracking-coding-agents-2026/).
The responses are institutional now: [real-time spend enforcement on
Bedrock](https://aws.amazon.com/blogs/machine-learning/tokenomics-at-scale-how-jamf-built-real-time-spend-enforcement-for-amazon-bedrock/),
[see, cap, and reduce agent spend as a shipped
feature](https://about.gitlab.com/blog/make-every-ai-credit-count/).

So the correction: the metered-everything argument is winning while nobody
watches the meter — and the meter is about to arrive. Most engineers have no
idea what their sessions cost; the lala land lasts until the first chargeback
table. The rest of the post stands.
