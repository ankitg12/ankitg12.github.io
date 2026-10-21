---
layout: post
title: "The catalog endpoint returns 37 ids and no prices. So I built the price list."
date: 2026-10-21
categories: ai agents tools
series: "AI coding agent productivity"
---

`GET https://opencode.ai/zen/go/v1/models` returns 37 model ids with `object`, `created`, `owned_by` — and nothing you need to pick one. No price, no context window, no output limit, no vision flag. The docs page has a pricing table, but tables don't answer "cheapest 1M-context model right now" at 11pm.

`opencode-go-models` is a single Go binary that layers the missing columns on:

```console
$ opencode-go-models --latest --sort output-price -n 10

OpenCode Go · 16 latest paid models · fetched 2026-10-21 15:31 UTC

  MODEL                       NAME                        CTX   OUT     IN/OUT $       CACHE $       VISION  FIRST SEEN
  muse-spark-1.3-contributor  Muse Spark 1.3 Contributor  1.0M  131.1K  $0.1/$0.2      $0.002/$0     👁       2026-10-21
  claude-haiku-5-5            Claude Haiku 5.5            100K  128K    $0.1/$0.5      $0.01/$0.125  👁       2026-10-21
  glm-5.3-flash               GLM-5.3-Flash               1M    131.1K  $0.15/$0.5     $0.03/$0      👁       2026-10-21
  gpt-6-luna                  GPT-6 Luna                  1.1M  128K    $0.1/$0.5      $0.01/$0.125  👁       2026-10-21
```

No key needed for the catalog. Usage windows need one:

```console
$ opencode-go-models --usage
  rolling    3%  (ok)
  weekly    82%  (ok)
  monthly   41%  (ok)
```

## Where the columns come from

Three sources, strict precedence:

| Layer | Source | Needs |
|---|---|---|
| id list | `GET /zen/go/v1/models` | nothing |
| context, limits, pricing, vision | local `omp models --json` when present | `omp` on PATH |
| usage windows | `GET /inference/go/v1/usage` | `--key` / `$OPENCODE_GO_API_KEY` |

The middle layer is optional by design. No `omp` on the machine, no key exported — you still get the id list plus cached enrichment. A missing source means fewer columns, never an error. The binary has zero coupling to any agent harness; anyone with the subscription can use it.

## The only honest "added on" date

Every row's `created` field is `1792595733` — identical across all 37 ids. Placeholder, not history. So the tool keeps its own: `first_seen`, set on first observation into `~/Notes/opencode-go-models.db`, never rewritten by later refreshes. After a few `--refresh` runs, `--sort seen` shows genuinely new arrivals. A fresh `--db` path resets history — expected, not a bug.

Cache TTL is 6h: a stale cache auto-refreshes with a stderr note, `--cached` never fetches, `--refresh` forces. The docs carry no release feed, so the cache is the change detector.

## --latest is a grouping problem

37 ids collapse to 16 families. The rule: group by vendor word (`gpt`, `grok`, `qwen3`, `mimo`), keep the highest version per group, stable beats preview. Two exceptions earned their tests: `kimi-k2` vs `kimi-k3` are distinct lines, and `qwen3.7-max` vs `qwen3.8-flash` are one family (`qwen3`). Numeric comparison alone picks wrong — `hy3` (3 > 0.4 on raw runs) would beat `hy4-preview`, so preview rows lose to same-family stable ids.

Filters compose: `--1m` (1M+ context), `--family qwen`, `--free` / `--all` (default hides zero-price rows), `--sort name|ctx|output|input-price|output-price|seen`, `-n`, `--json`, `--save`, `--sql` for read-only queries against the cache.

The footer counts exactly what it hid, because a lying footer cost a commit five minutes after the first demo:

```console
10 shown · 27 hidden (4 free/preview, 17 filtered by --latest/--family/--1m, 6 cut by -n)
```

## Source

- [ankitg12/opencode-go-models](https://github.com/ankitg12/opencode-go-models) — the tool
- [OpenCode Go docs](https://opencode.ai/docs/go/) — plans, per-model pricing, monthly limits
- [OpenCode console models](https://opencode.ai/v2/docs/console/models/) — pay-as-you-go reference
- [Lordymine/opencode-go-cli](https://github.com/Lordymine/opencode-go-cli) — prior art: live fetch + 1h cache, ids only
