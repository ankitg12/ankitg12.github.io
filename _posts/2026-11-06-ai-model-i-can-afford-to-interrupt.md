---
layout: post
title: "The Value of an AI Model I Can Afford to Interrupt"
date: 2026-11-06
categories: ai agents productivity
series: "AI coding agent productivity"
---

I have been working with a lower-cost AI model on a systems engineering task. As we worked, I noticed something I had not accounted for when comparing models: I was more willing to interrupt it.

I could correct its approach, comment on its output, and discuss a different direction without worrying as much about the cost of every exchange. That gave me more opportunities to shape the work while it was happening.

My estimate is that a similar session with a frontier model could have cost $30–$40 or more. I have not run a controlled comparison, so that is an estimate, not a measured saving.

The observation is about my behavior: **when feedback feels affordable, I give more of it.**

## The first answer is not the whole job

A strong model can produce an impressive first result. But it can also make assumptions I would have challenged, choose a direction I did not intend, or complete a lot of work before I get a chance to respond.

By the time I review the result, correcting it may mean revisiting the whole approach.

During this session, I could intervene earlier. I could point out a formatting problem, question a choice, or redirect the work without feeling that each exchange needed to justify an expensive model call.

That did not make the model infallible. It made mistakes, and some of the engineering work remains blocked. But it changed the working relationship. I had more room to participate.

## The value of a feedback loop

When I compare models, I often focus on the first attempt: which model produces the best answer from the same prompt.

I also want to consider the whole task: how much feedback I can give within the budget, how soon I can catch a wrong assumption, how much time I spend reviewing and correcting, and whether the final result meets the acceptance criteria.

A weaker first attempt is not always a bad trade if I can inspect it, correct it, and reach a sound result. But more iterations are not automatically better. They cost time and attention, even when the model calls are inexpensive.

## Others have seen parts of this

Daniel Schötz describes building a game with a free, less capable model through a repeated **build, test, critique, and improve** loop. He says the model made more mistakes and needed more guidance, but iteration made progress possible. That is one project, not proof that a weaker model will be better for every task. [1]

AJ Fisher makes a related argument about speed: once a model is capable enough for a task, a faster feedback loop can matter more than a stronger first attempt. His experiment uses automated tests; it does not measure my willingness to give feedback. [2]

Sami Honkonen describes using a frontier model as an “architect” and a cheaper model as an “engineer.” He keeps the human involved in shaping decisions. His approach is different from mine, but it reflects the idea that different parts of a task can need different models. [3]

Arda Hoke reports reducing the cost of a coding-agent workflow by routing work across models. He also notes that comparing finding quality was difficult and recommends measuring cost per accepted task, including retries and human review. [4]

## Cheap calls are not free work

A lower-priced model can take more attempts, miss a requirement, or require more supervision. If I spend an hour correcting work that a stronger model would have completed reliably, the lower API bill may not be a saving.

A fair comparison needs to include the model cost, failed attempts, elapsed time, my review time, and the quality of the accepted result. It should also record interventions: when I stepped in and what changed because I did.

I do not yet have that comparison. For now, my claim is narrower: **a lower model cost made me feel freer to give feedback during the work.**

## Choosing the working relationship

I am not giving up on stronger models. There are tasks where their judgment, breadth, or ability to solve a problem with fewer attempts may be worth the cost. They can also be useful for reviewing work done by a cheaper model.

But I do not want to select a model only by the quality of its first response. I want to consider whether I can afford to stay involved, correct its direction, and help shape the result.

Sometimes the model I can afford to interrupt may be the better collaborator.

## Sources

1. Daniel Schötz, [You Don’t Need the Best Model. You Need a Better Loop](https://www.phooky.com/blog/you-Do-not-need-the-best-model-you-need-a-better-loop/), 2026.
2. AJ Fisher, [Fail fast, fix faster: Why faster models can beat smarter ones](https://ajfisher.me/2026/06/13/fail-fast-fix-faster/), 2026.
3. Sami Honkonen, [Two models are cheaper than one](https://samihonkonen.com/posts/two-models-are-cheaper-than-one/), 2026.
4. Arda Hoke, [How I cut coding agent costs with model and harness routing](https://arize.com/blog/how-i-cut-coding-agent-costs-with-model-harness-routing/), 2026.
