---
layout: post
title: "Running Hindsight as OMP's Memory Backend on Windows"
date: 2026-08-12 22:45:00 +0530
categories: windows ai agents productivity memory
series: "AI coding agent productivity"
---

An agent reported that it had learned something, yet the memory never reached durable storage.

```text
Warning: Hindsight: Memory retention failed
retainBatch request failed: Unable to connect
```

The failure was not in memory extraction. OMP was configured to send the batch to a Hindsight API that did not exist on the configured port. A second local service made the diagnosis less obvious: the model adapter was healthy at one point, but it was not the memory API.

This post documents the working Windows topology, the bootstrap traps, and the verification sequence that distinguishes “queued” from “retained.”

---

## The topology: two services, two responsibilities

```text
OMP
 └─ retainBatch ────────────────> Hindsight API :8888
                                      │
                                      ├─ local embeddings + reranker
                                      ├─ embedded PostgreSQL/pgvector
                                      │
                                      └─ OpenAI-compatible request ──> adapter :8788
                                                                            │
                                                                            └─ enterprise LLM gateway
```

| Port | Service | Responsibility |
|---:|---|---|
| 8888 | Hindsight API | Receives OMP memory batches; extracts, stores, and recalls facts |
| 8788 | Optional OpenAI-compatible adapter | Translates Hindsight's model request into the authentication/header convention required by an upstream gateway |

The adapter is optional in the general case. If the model endpoint accepts standard OpenAI Bearer authentication, Hindsight can call it directly. I needed the adapter because my upstream endpoint requires a custom authentication header that Hindsight's OpenAI provider does not currently emit.

The important diagnostic rule is simple:

> A listening model adapter does not prove that OMP can reach Hindsight.

Resolve and probe the memory API first.

### LiteLLM as the authentication adapter

The adapter is [LiteLLM](https://github.com/BerriAI/litellm), running as an OpenAI-compatible localhost proxy. Its configuration is deliberately small:

```yaml
model_list:
  - model_name: memory-model
    litellm_params:
      model: openai/<upstream-model-name>
      api_base: https://<enterprise-gateway>/v1
      api_key: dummy
      extra_headers:
        Ocp-Apim-Subscription-Key: os.environ/UPSTREAM_LLM_KEY

litellm_settings:
  drop_params: true
```

Start it on loopback:

```powershell
$env:UPSTREAM_LLM_KEY = "<secret>"
litellm --config .\litellm-hindsight.yaml --host 127.0.0.1 --port 8788
```

Hindsight then uses:

```text
HINDSIGHT_API_LLM_PROVIDER=openai
HINDSIGHT_API_LLM_MODEL=memory-model
HINDSIGHT_API_LLM_BASE_URL=http://127.0.0.1:8788/v1
HINDSIGHT_API_LLM_API_KEY=dummy
```

The dummy client key is acceptable here only because this specific adapter does not validate downstream Bearer tokens; loopback limits network exposure but does not make an unauthenticated service intrinsically safe from other local processes. Apply host firewall and local-user controls appropriate to the machine. The real upstream credential stays in the proxy process environment rather than the LiteLLM YAML or Hindsight profile.

---

## 1. Confirm OMP's resolved configuration

Do not infer the endpoint from a config fragment. Ask OMP for its resolved configuration:

```powershell
omp config list --json
```

The relevant values are:

```text
memory.backend = hindsight
hindsight.apiUrl = http://localhost:8888
```

On an agent sandbox, `HOME` may not be the interactive user's Windows profile. Set `HOME`, `USERPROFILE`, and the OMP agent directory explicitly only for commands that resolve user-relative configuration; localhost HTTP probes do not need that boilerplate.

Probe both layers independently:

```powershell
8788, 8888 | ForEach-Object {
    [pscustomobject]@{
        Port      = $_
        Listening = (Test-NetConnection 127.0.0.1 -Port $_ `
            -WarningAction SilentlyContinue).TcpTestSucceeded
    }
}
```

---

## 2. Install the supported Windows embedded runtime

Hindsight's native Windows path is `hindsight-embed`, distributed through `uvx`:

```powershell
uvx hindsight-embed profile create omp `
    --port 8888 `
    --env HINDSIGHT_API_LLM_PROVIDER=openai `
    --env HINDSIGHT_API_LLM_MODEL=<model-name> `
    --env HINDSIGHT_API_LLM_BASE_URL=http://127.0.0.1:8788/v1 `
    --env HINDSIGHT_API_LLM_API_KEY=dummy
```

Use a real secret when the target endpoint validates Bearer tokens. `dummy` is appropriate only when a trusted localhost adapter terminates the client request and supplies upstream authentication itself.

Start the profile:

```powershell
uvx hindsight-embed -p omp daemon start
```

### First boot is not a quick health check

The first start creates an isolated Python environment, downloads the embedded database and local embedding/reranker models, initializes PostgreSQL, and applies schema migrations. It can take several minutes.

Two observations prevent wasted work:

1. `uv` uses a shared cache. Re-reading an append-only startup log can make one continuing download look like repeated installations.
2. A foreground shell timeout does not prove that child processes stopped. Inspect process command lines and the profile log before launching another bootstrap.

A healthy startup eventually reports:

```text
HTTP REST API enabled
Starting Hindsight API...
URL: http://127.0.0.1:8888
Database migrations completed successfully
```

---

## 3. Supervise the API process

For an agent runtime, the useful property is not “daemonized”; it is **observable lifecycle ownership**. Run the API under the same process supervisor that owns the agent session, and declare port 8888 as its readiness condition.

This prevents three ambiguities:

- a detached child outliving the credential context that created it;
- a dead process leaving only an append-only historical log;
- “process exists” being mistaken for “API accepts connections.”

Keep the model adapter similarly bounded when it carries a gateway credential. Do not make a credential-bearing shim persistent merely to hide missing session provisioning.

---

## 4. Verify retain and recall directly

A successful agent-side `learn` call may mean only **queued for asynchronous retention**. Verify the storage system itself before replaying valuable facts.

### Synchronous retain

```powershell
$body = @{
    items = @(@{
        content = "Verification token HINDSIGHT-E2E-001"
        context = "end-to-end setup verification"
    })
    async = $false
} | ConvertTo-Json -Depth 5

Invoke-RestMethod `
    -Method Post `
    -Uri "http://127.0.0.1:8888/v1/default/banks/verification/memories" `
    -ContentType "application/json" `
    -Body $body
```

Expected evidence:

```json
{
  "success": true,
  "bank_id": "verification",
  "items_count": 1,
  "async": false
}
```

### Recall

```powershell
$body = @{
    query = "What is the end-to-end verification token?"
    budget = "low"
} | ConvertTo-Json

Invoke-RestMethod `
    -Method Post `
    -Uri "http://127.0.0.1:8888/v1/default/banks/verification/memories/recall" `
    -ContentType "application/json" `
    -Body $body
```

The result must contain the token. A listening port, a queued tool response, or the absence of an immediate warning is weaker evidence.

---

## 5. Recover failed batches without inventing memories

A failed asynchronous batch may no longer contain a replayable payload. Recovery therefore has two classes:

| Class | Recovery |
|---|---|
| Explicit learning call preserved in the session transcript | Replay the exact content after direct retain/recall passes |
| Automatically extracted fact absent from logs | Reconstruct a candidate from the source conversation and review it before retaining |

Keep disproven facts out. In this incident, an early diagnosis incorrectly treated the model adapter as the `retainBatch` endpoint. That payload was discarded and replaced by the corrected two-service fact.

After replay:

1. require the Hindsight operation log to report completion;
2. recall each fact from OMP's actual bank;
3. confirm unrelated local artifacts—such as managed skills—did not change as a side effect.

The last check matters when one command can both retain a memory and update a skill. A failed asynchronous retain does not imply that the synchronous skill mutation also failed.

---

## A more reliable operating pattern

The incident suggests a transactional-outbox pattern for agent memory:

1. write the candidate fact to a durable local outbox;
2. assign a stable local delivery identifier;
3. submit it and mark it delivered only after the memory API acknowledges completion;
4. retain failed payloads for bounded replay;
5. verify high-value facts through recall.

This installation did not implement that outbox. Before adding automatic retries, verify the installed Hindsight version's deduplication semantics; replay without an idempotency guarantee can create duplicate memories. Even a manual outbox would turn a transient service outage from silent fact loss into a reviewable delivery problem.

The distinction is the same one used in distributed systems generally: **accepted locally is not committed remotely**.

---

## Source

- [Hindsight installation](https://hindsight.vectorize.io/developer/installation)
- [Hindsight configuration](https://hindsight.vectorize.io/developer/configuration)
- [Hindsight model providers](https://hindsight.vectorize.io/developer/models)
- [Hindsight HTTP API quick start](https://hindsight.vectorize.io/developer/api/quickstart)
- [Hindsight source repository](https://github.com/vectorize-io/hindsight)
- [LiteLLM source repository](https://github.com/BerriAI/litellm)
- [Oh My Pi source repository](https://github.com/can1357/oh-my-pi)
