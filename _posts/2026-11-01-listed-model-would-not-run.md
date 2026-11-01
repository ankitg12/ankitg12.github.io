---
layout: post
title: "The Model Listed Fine. It Just Wouldn't Run."
date: 2026-11-01
categories: ai agents debugging
series: "AI coding agent productivity"
---

`openclaw models list --provider opencode-go` showed `muse-spark-1.3-contributor` as the default. The webchat answered with `Unknown model: opencode-go/muse-spark-1.3-contributor`. Same model, same machine, two different answers.

The listing path and the runtime path read different catalogs.

## Two catalogs, one name

The [list command](https://github.com/openclaw/openclaw/issues/142793) reads the live endpoint at `GET https://opencode.ai/zen/go/v1/models` — 28 models on the night I hit this. The runtime resolver reads a static seed catalog bundled with the plugin — 7 models (`deepseek-v4-pro`, `deepseek-v4-flash`, `gpt-5.6-luna`, `hy3-preview`, `kimi-k2.6`, `kimi-k3`, `qwen3.8-max`). Spark is live-only, so it lists and then dies at first turn:

```
Unknown model: opencode-go/muse-spark-1.3-contributor. Run
`openclaw models list --refresh --provider opencode-go` to inspect
this provider's model choices, then retry with a model supported by
your account.
```

The suggested command changes nothing — refresh repopulates the list catalog, not the seed the runner consults.

The proof that two catalogs are involved, from my own shell:

```console
$ openclaw models list --provider opencode-go | grep spark
opencode-go/muse-spark-1.3-contributor  text+image  1049k  no  yes  default

$ openclaw infer model inspect --model opencode-go/muse-spark-1.3-contributor --json
{"ok": false, "error": {"type": "cli_error",
  "message": "Model not found: opencode-go/muse-spark-1.3-contributor"}}

$ openclaw infer model list --json | grep -c opencode-go
0
```

11 rows, all `claude-cli:*`. Zero `opencode-go:*`. And the gateway log confirmed the split — deepseek (a seed model) returned HTTP 200 minutes before spark failed with `status=format`:

```
provider=opencode-go model=deepseek-v4-pro status=200
Unknown model: opencode-go/muse-spark-1.3-contributor ... runner=embedded ... status=format
```

## The workaround: one `models-add` row

The [upstream issue](https://github.com/openclaw/openclaw/issues/142793) documents the escape hatch: declare the missing model under `models.providers` with `metadataSource: "models-add"`. It also warns that a non-empty `models[]` entry [shadows the plugin catalog](https://github.com/openclaw/openclaw/issues/165414) on some builds — 29 models collapsing to 10. On my build (2026.10.9) that didn't happen; the list stayed full. A [fix for local turns](https://github.com/openclaw/openclaw/pull/168618) merged 2026-10-11, two days after my build — the gateway path I use still fails without the patch.

I only use spark, so the patch is one row plus runtime mappings, in `~/.openclaw/openclaw.json`:

```json5
{
  models: {
    mode: "merge",
    providers: {
      "opencode-go": {
        models: [
          { id: "muse-spark-1.3-contributor", name: "Muse Spark 1.3 Contributor",
            input: ["text", "image"], contextWindow: 1048576,
            metadataSource: "models-add" },
        ],
      },
    },
  },
  agents: {
    defaults: {
      model: "opencode-go/muse-spark-1.3-contributor",
      models: {
        "opencode-go/muse-spark-1.3-contributor": { agentRuntime: { id: "openclaw" } },
      },
    },
  },
}
```

Both pieces matter. The `models-add` row teaches the resolver the model exists. The `agentRuntime` mapping in `agents.defaults.models` (and `agents.entries.main.models`) tells the runner which harness executes it. I learned this the asymmetric way — the first patch had the entry but the old `deepseek-v4-pro` mapping, and the default flipped without resolving.

Applied via validated patch, no gateway restart:

```console
$ openclaw config patch --file spark-patch.json5 --dry-run --json
{"ok": true, "operations": 5, "checks": {"schema": true, "resolvability": true}}

$ openclaw agent --message "Reply with the word OK only." --json
... "winnerModel": "muse-spark-1.3-contributor", "result": "success" ...
"terminalReply": {"text": "OK"}
```

| Step | Command | Before | After |
|---|---|---|---|
| list | `models list --provider opencode-go` | spark shown, `default` | spark shown, `default,configured` |
| inspect | `infer model inspect --model ...` | `Model not found` | still seed-only (expected — different catalog) |
| turn | `agent --message ...` | `Unknown model`, `status=format` | `result: success`, reply `OK` |

## Why not upgrade instead

The [October 11 fix](https://github.com/openclaw/openclaw/pull/168618) covers local `agent --local` turns for credential-only providers. My failure is on the gateway path (`runner=embedded` through `ws://127.0.0.1:18789`), which that PR's proof doesn't exercise the same way — its trace is a separate `agent --local` process with its own SQLite cache. Upgrading might fix it, might not; the one-row patch is five operations, validated dry-run first, backup kept at `~/.openclaw/openclaw.json.bak-spark-patch`, and it provably works on the exact build I run.

## Why not declare all 28 models

The live catalog had 28 ids the night I patched. Declaring all of them means re-running the fetch on every new release or hitting the same wall next month. I use one model; one row. Anyone who rotates models should script the fetch-to-patch step — my [opencode-go-models](https://github.com/ankitg12/opencode-go-models) tool already layers [models.dev](https://models.dev) pricing and context columns onto the same endpoint, so generating the full `models[]` block from live data is one `--json` flag away from done.

Later: re-test the gateway path after upgrading past 2026.10.11, then delete the patch if the seed/live split is gone. Until then the row stays.

## Source

- [openclaw/openclaw#142793](https://github.com/openclaw/openclaw/issues/142793) — `Unknown model` at runtime for models that list fine (root cause: seed-only resolver)
- [openclaw/openclaw#165414](https://github.com/openclaw/openclaw/issues/165414) — follow-up: non-empty `models[]` shadows the plugin catalog (open)
- [openclaw/openclaw#168618](https://github.com/openclaw/openclaw/pull/168618) — fix: refreshed models work in local agent turns (merged 2026-10-11)
- `GET https://opencode.ai/zen/go/v1/models` — the live endpoint the list path reads
- [models.dev](https://models.dev) — pricing/context snapshot for opencode-go rows
- [ankitg12/opencode-go-models](https://github.com/ankitg12/opencode-go-models) — catalog + price CLI for the same endpoint
