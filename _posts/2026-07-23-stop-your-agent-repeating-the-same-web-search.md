---
layout: post
title: "Stop your agent repeating the same web search"
date: 2026-07-23
updated: 2026-07-26
categories: ai agents productivity
series: "AI coding agent productivity"
---

My coding agent runs a periodic "retro" every ten minutes, and one of its steps is a treat: *search the web for a tool or technique adjacent to what you just did, to keep learning.* Lovely idea. Except after a few months I noticed I was being shown the same handful of "gems" over and over — chezmoi encryption, gitleaks hooks, the same three agent-orchestration tools — as if the agent had a favourite orbit it couldn't escape.

It couldn't. The step said "search something adjacent you don't already know," but nothing anywhere recorded what had *already* been surfaced. No memory, so no novelty. The "adjacent" space around clustered work collapses to the same dozen topics and repeats forever.

## Diagnose before building

Before writing a line of tool, I mined my agent's own history. The session viewer keeps every tool call in a local SQLite DB, so the actual questions are answerable rather than guessable:

```sql
SELECT tool_name, input_json FROM tool_calls
WHERE tool_name IN ('web_search','WebSearch','search');
```

That was **4,938 real search calls over ~6 months**. Normalising the query strings and clustering by keyword gave the smoking gun — a long tail of one-offs, and a head of the same topics resurfacing across dozens of unrelated sessions. Exact-string repeats topped out low; the *topic-level* repetition was the real waste. chezmoi showed up 30 times, gitleaks 25.

Lesson one: **your agent already logs the evidence for its own bad habits.** Query it before theorising.

## The fix is a gate, not a document

The naive fix is "append every gem to a markdown file and tell the agent to read it first." That fails twice: the file grows unbounded into the context window (the exact bloat you're trying to avoid), and *reading* a list doesn't *prevent* a repeat — it just hopes the model notices.

The deterministic fix is a **gate**. A tiny SQLite ledger with one job: when the agent proposes a topic, either record it (novel) or refuse it (already covered). Only the refusal returns to context, so a repeat costs one short line instead of the whole history:

```bash
$ gemsearch.py search "gitleaks hook for secrets"
DUPLICATE - already covered: #42 gitleaks pre-commit hook  [2026-05-11]
Pick a genuinely new/farther-afield topic and search again.   # exit 2
```

One command does the whole loop: gate for novelty → if novel, run the actual web search → store the results → print them. If it exits non-zero, the agent picks a farther-afield topic and calls again. The template step shrinks to a single line, and the *tool* owns the behaviour instead of a paragraph of prompt.

## What the dedup should (and shouldn't) do

My first matcher was too clever. I weighted rare tokens as "anchors" and blocked on token overlap — and promptly rejected *"io_uring zero-copy networking"* as a duplicate of *"named-pipe SSH multiplexing"* because they shared the common word "internals." Over-blocking is worse than the disease: it silently starves you of genuinely new topics.

So the shipped matcher is deliberately **conservative** — it only blocks on high-confidence signals:

| Signal | Example it catches |
|---|---|
| identical normalised signature | `gitleaks hook` vs `Gitleaks Hook` |
| subset of tokens either way | `chezmoi` vs `chezmoi encryption` |
| sequence-similarity ≥ 0.82 | `git absorb` vs `git-absorb` |

It will *miss* a semantic rephrase that shares few words (`SONiC architecture` vs `SONiC dataplane deep-dive`). Catching those needs embeddings — and the honest, boring, dependency-free way to approximate it is **SimHash / locality-sensitive hashing**, which is the planned upgrade rather than dragging in a vector database for two thousand rows.

That last point is the real lesson. I *did* start dragging in embeddings and a vector DB, and had to strip it all back out. For a two-thousand-row personal ledger, a distributed vector store is theatre. Boring SQLite plus a conservative string match is the correct amount of engineering — and it's the version that shipped, works, and never blocks a real idea.

## The bug the conservative matcher missed

Three days later, the agent searched for *"human factors progressive disclosure discoverability command palettes"* and presented the same progressive-disclosure articles it had already found for *"feature flags configuration progressive disclosure extension UI"*.

The query gate behaved exactly as designed—and still failed the user. The two normalised signatures shared only `progressive` and `disclosure`; their `SequenceMatcher` ratio was **0.47**, well below the **0.82** threshold. Tightening the wording matcher would recreate the false-positive problem above. More importantly, it would optimize the wrong thing: I cared whether the agent repeated the same **knowledge**, not whether it repeated the same **query**.

The fix adds a second gate after the web search:

1. Extract and canonicalise result URLs, ignoring local proxy links.
2. Compare the candidate URL set with stored result sets.
3. Reject the search when at least three sources recur, or when at least two recur and make up at least 35% of the smaller result set.

The repeated search shared IxDF, UX/UI Principles, and Lollypop Design, so it now exits `2` and points to the earlier ledger entry instead of recording another gem:

```text
DUPLICATE - search results already covered by:
#1969 feature flags configuration progressive disclosure extension UI [2026-07-25]

Pick a genuinely new/farther-afield topic and search again.
```

Four regression tests defend the boundary: substantial two-source overlap and any three-source overlap are duplicates; one shared source is not; and a rejected search never inserts another ledger row.

This is a useful distinction for any retrieval system: **deduplicating requests is not the same as deduplicating outcomes**. Query similarity remains the cheap pre-search gate; source overlap is the evidence-based post-search gate. Embeddings are still an available later layer for cases where different sources restate the same knowledge, but they were not required to fix the observed failure.

## Why not MinHash or RETSim?

The adjacent research turned up two credible alternatives. Both are good techniques; neither matches the failure we actually observed.

| Approach | What it compares | Strength | Cost or blind spot | Fit here |
|---|---|---|---|---|
| Exact canonical URL sets (current) | The 5–10 sources returned by each search | Deterministic, lossless, explainable, standard-library only | Misses equivalent knowledge hosted at different URLs | Best fit: the failure repeated three identical sources |
| MinHash, usually with LSH | Compact signatures that approximate Jaccard similarity between large sets | Fast candidate retrieval across very large collections | Probabilistic; signature size controls error; still needs meaningful set elements | Premature: exact intersection over roughly 2,000 tiny URL sets is cheaper and has no estimation error |
| RETSim | Learned character-level embeddings of document text | Robust near-duplicate retrieval across typos, edits, languages, and adversarial perturbations | Requires fetching text, model inference, an index, and calibrated similarity thresholds | Solves a harder problem we have not yet observed: different sources carrying near-duplicate text |

### Jaccard similarity, from first principles

Jaccard similarity answers a simple question: **of all distinct items seen across two sets, what fraction appears in both?** Divide the size of the intersection—the shared items—by the size of the union—all distinct items:

```text
Jaccard(A, B) = |A ∩ B| / |A ∪ B|
```

Suppose one search returns `{A, B, C, D, E}` and another returns `{A, B, C, F, G}`. They share three sources; together they contain seven distinct sources. Their Jaccard similarity is `3 / 7 ≈ 0.43`. A score of `0` means no overlap; `1` means the sets are identical.

Our current gate deliberately uses a slightly different measure for the two-source rule: `shared / size of smaller set`, sometimes called the **overlap coefficient** or containment score. If a narrow three-result search repeats two sources from a broad ten-result search, Jaccard is only `2 / 11 ≈ 0.18`, but containment is `2 / 3 ≈ 0.67`—a better signal that most of the narrow search is recycled. The unconditional three-source rule then catches substantial recurrence even when both result lists are long. MinHash naturally estimates Jaccard; reproducing our asymmetric containment policy with sketches would require additional machinery.

MinHash is particularly elegant when exact set comparison becomes the bottleneck. With `k` hash functions it represents each set using a fixed-size signature and estimates Jaccard similarity from signature agreement; the expected error falls as `O(1/√k)`. Combined with locality-sensitive hashing, it can avoid scanning every historical set. But our sets contain only a handful of URLs. Computing their exact intersection preserves the same signal without approximation, tuning, or another dependency.

RETSim addresses a different layer. The published model uses a character-level vectorizer plus a small transformer—536K parameters—to produce 256-dimensional embeddings for robust near-duplicate text retrieval. It substantially outperforms MinHash on the paper's multilingual and adversarially modified text benchmarks. To use it here, however, `gemsearch` would need to download and normalize article bodies, run inference, store embeddings, and decide a threshold. That is defensible only after observing repeats where the URLs differ but the underlying content is genuinely the same.

The decision rule is therefore evidence-led:

```text
same query wording?       → deterministic pre-search string gate
same returned sources?    → deterministic post-search URL-set gate
same content, new URLs?   → add RETSim only after this failure appears
millions of stored sets?  → consider MinHash + LSH when exact scans become measurable
```

This is not reluctance to use sophisticated machinery. It is architectural discipline: **choose the cheapest representation that preserves the signal you need, and promote complexity only when measured false negatives demand it.** For an engineer building toward Fellow-level scope, the important skill is not knowing that MinHash and RETSim exist; it is being able to explain precisely why the production boundary does—or does not—justify them.

## Takeaways

- **Query your agent's own logs** before building; the evidence is already there.
- **Gate, don't document** — a rejection costs one line; a re-read costs the whole history.
- **Conservative dedup needs two levels** — compare the proposed query before searching, then compare the returned evidence before recording it.
- **Right-size the storage** — SQLite and a string match, not a vector DB, until the data actually demands more.

## Source

- [`gemsearch.py` (gist)](https://gist.github.com/ankitg12/1c1d19ac81190d2b1d5264fa6b2ca367) — original dependency-free implementation; the post-search URL-overlap gate above is the subsequent fix
- [MinHash](https://en.wikipedia.org/wiki/MinHash) — approximate Jaccard similarity for large-scale set comparison
- [RETSim: Resilient and Efficient Text Similarity](https://arxiv.org/abs/2311.17264) — lightweight learned embeddings for robust near-duplicate text retrieval
- [GPTCache](https://github.com/zilliztech/GPTCache) — semantic caching done the heavyweight way, for when you genuinely need it
