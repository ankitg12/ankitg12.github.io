---
layout: post
title: "A Cost Circuit Breaker for Long-Running OMP Sessions"
date: 2026-08-06
categories: ai agents productivity
series: "AI coding agent productivity"
---

A productive coding-agent session can quietly become a surprisingly expensive one.

The footer may display cumulative cost, but that still depends on a human noticing it while concentrating on the actual problem. The safer pattern is the same one used in electrical systems: warn before the expected operating range ends, then trip a circuit breaker when the limit is reached.

I wanted this policy:

```text
$8   warn once
$10  abort once and require a deliberate continuation
$15  remind
$20  remind
$25  remind
...
```

The result is [`agent-cost-guard-omp`](https://github.com/ankitg12/agent-cost-guard-omp), a zero-dependency extension for OMP.

## Why not end every session at exactly $10?

A dollar boundary is not necessarily a context boundary.

Starting a fresh session discards irrelevant history, but it can also discard a useful working model and provider-side prompt-cache benefits. The better rule is:

|Condition|Action|
|---|---|
|The task or direction changes|Start a fresh session|
|Accumulated context becomes confusing|Start a fresh session|
|Cost reaches the warning threshold|Assess whether the context still earns its keep|
|Cost reaches the limit|Stop once and make continuation deliberate|
|Work deliberately continues|Remind at fixed additional-cost checkpoints|

The limit is therefore a circuit breaker, not a session-design philosophy.

## Install it

Clone the repository, then add the extension file to `~/.omp/agent/config.yml`:

```yaml
extensions:
  - ~/repos/github.com/ankitg12/agent-cost-guard-omp/cost-guard.ts
```

The defaults require no additional configuration. They can be overridden per launch:

```bash
OMP_COST_WARN=8 \
OMP_COST_LIMIT=10 \
OMP_COST_REMINDER_STEP=5 \
omp
```

## Count the authoritative session branch

Incrementing a local counter on each new response looks sufficient, but it misses two important cases:

1. a resumed session already contains earlier spend;
2. subagent work returned by OMP's `task` tool carries its own usage.

The extension instead recomputes spend from the active session branch:

```ts
function sessionCost(sessionManager: ReadonlySessionManager): number {
  let spent = 0;

  for (const entry of sessionManager.getBranch()) {
    if (entry.type !== "message") continue;

    const message = entry.message;
    if (message.role === "assistant") {
      spent += usageCost(message.usage);
    }

    if (message.role === "toolResult" && message.toolName === "task") {
      spent += usageCost(Reflect.get(message.details, "usage"));
    }
  }

  return spent;
}
```

Recomputing from authoritative state is deliberately boring. It avoids synchronizing another cost ledger.

## Make reminders idempotent

This expression identifies the latest five-dollar checkpoint:

```ts
const threshold = Math.floor(spent / reminderStep) * reminderStep;
```

It is not sufficient by itself. Once spend is above `$15`, it evaluates to `$15` on every subsequent turn until `$20`; without state, the same reminder fires repeatedly.

The extension retains the last emitted checkpoint:

```ts
if (threshold > lastReminded) {
  lastReminded = threshold;
  notify(`Session cost is $${spent.toFixed(2)}`);
}
```

That small state variable makes the notification edge-triggered rather than level-triggered.

## Resume after the breaker fires

OMP's `ctx.abort()` stops the current agent operation; it does not delete the persisted session. A later message—or a resumed process—can continue from the same history.

On `session_start`, the extension reads existing spend and initializes its state:

```ts
pi.on("session_start", (_event, ctx) => {
  const spent = sessionCost(ctx.sessionManager);
  warned = spent >= warnAt;
  tripped = spent >= limit;
  lastReminded = Math.floor(spent / reminderStep) * reminderStep;
});
```

This matters because resuming an acknowledged `$12` session should not immediately repeat the `$10` abort. It should continue toward the `$15` reminder.

## The unavoidable soft-cap limitation

This extension learns actual cost at `turn_end`, after the model provider has completed and billed the response. Consequently:

```text
$9.80 before request
+$1.40 response
----------------
$11.20 when the guard can react
```

The guard prevents subsequent work, but cannot recall tokens already generated. A single expensive turn can overshoot the nominal limit.

A hard cap requires enforcement before request dispatch. Ideally OMP would support:

```bash
omp --warn-cost 8 --max-cost 10
```

Native implementation can use OMP's authoritative accounting at the model-dispatch boundary and apply the same policy to interactive, print, resumed, and subagent execution paths. I opened [oh-my-pi issue #7802](https://github.com/can1357/oh-my-pi/issues/7802) for that capability.

## Keep the mechanism smaller than the problem

The extension has three numbers and one job:

```text
warn once → abort once → remind periodically
```

It does not contain a dashboard, a pricing database, per-model policy, forecasting, or billing persistence. Those belong in OMP itself or an LLM gateway with pre-flight admission control.

The useful outcome is not perfect cost prediction. It is converting unnoticed spend into an explicit decision point.

## Source

- [`agent-cost-guard-omp`](https://github.com/ankitg12/agent-cost-guard-omp)
- [OMP extension documentation](https://github.com/can1357/oh-my-pi/blob/main/docs/extensions.md)
- [Feature request: native `--warn-cost` and `--max-cost`](https://github.com/can1357/oh-my-pi/issues/7802)
- [Token-Budget-Aware LLM Reasoning](https://arxiv.org/abs/2412.18547)
