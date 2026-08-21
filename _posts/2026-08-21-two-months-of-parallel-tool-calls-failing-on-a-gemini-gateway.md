---
layout: post
title: "Two Months of Parallel Tool Calls Failing on a Gemini Gateway"
date: 2026-08-21
categories: ai agents debugging
series: "AI coding agent productivity"
---

Some bugs are hard because the failure is complicated. This one was hard because the failure was *simple* and every layer of the stack lied about who caused it.

The symptom, roughly every time I tried to use a Gemini model through an OpenAI-compatible enterprise gateway inside a coding agent:

```text
Please ensure that the number of function response parts is equal to the
number of function call parts of the function call turn.
```

The agent had just sent two tool calls and two tool results. Two equals two. The gateway disagreed, and had been disagreeing, on and off, for two months.

Today it works. The fix is about thirty lines of extension code, and none of the thirty lines are the interesting part. The interesting part is why four earlier, more reasonable-looking approaches all failed.

---

## The shape of the problem

Gemini's function calling carries an opaque **thought signature**: a provider-side blob that ties a function call back to the model's internal reasoning state for that turn. Google documents these as something you must return unmodified on the follow-up request ([Gemini thinking docs](https://ai.google.dev/gemini-api/docs/thinking)).

The OpenAI Chat Completions wire format has no field for that. So the gateway smuggles it through the one field that survives a round trip: the tool call ID.

```text
call_9b3ee4c835c649c1bf1||thought_signature||AY89a19edySm1kK...
```

That is a beautiful hack and it works, right up until two things happen at once.

**Thing one:** clients truncate IDs. Many clients treat a tool call ID as an opaque short token, normalize it, regenerate it, or cap its length. If any layer rewrites that string, the signature is gone and the next request is rejected.

**Thing two — and this is the killer:** in a parallel tool-call turn, **only the first call carries a signature.**

```text
call_9b3ee4c835c649c1bf1||thought_signature||AY89a19edySm1kK...   ← signed
call_f2f992162b3f4950b02                                          ← bare
```

So when the client faithfully replays two calls and two results, the gateway's translation layer rebuilds the Gemini-side model turn from what it can attribute — one signed call — and then sees two function responses. One call part, two response parts. Rejected.

The error message describes the *post-translation* state. The client never sees that state. You are debugging a mismatch that does not exist in the payload you are looking at.

---

## Attempt 1: a local proxy that fixes the headers

The first fix was not even about signatures. The gateway authenticates with a custom header, while many OpenAI-compatible clients only know how to send `Authorization: Bearer`. A [LiteLLM](https://docs.litellm.ai/docs/proxy/configs) config terminates the bearer call and re-issues it with the right header:

```yaml
model_list:
  - model_name: gemini-pro
    litellm_params:
      model: openai/gemini-pro
      api_base: https://gateway.internal/v1
      api_key: dummy
      extra_headers: { "X-Subscription-Key": "os.environ/GATEWAY_KEY" }
      parallel_tool_calls: false
litellm_settings:
  drop_params: true
```

This solved authentication completely. It solved tool calling not at all.

Note the hopeful `parallel_tool_calls: false` on line ten. It does nothing here. `drop_params: true` may strip it, the gateway may ignore it, and the model may emit parallel calls regardless. I would go on to re-learn this lesson twice more.

## Attempt 2: a proxy callback that restores stripped signatures

If clients truncate IDs, teach the proxy to remember them. A LiteLLM `CustomLogger` that watches responses, caches every full `id||thought_signature||sig` it sees, and rewrites truncated IDs back to their full form on the next request:

```python
_DELIMITER = "||thought_signature||"

async def async_pre_call_hook(self, user_api_key_dict, cache, data, call_type):
    for message in data.get("messages", []):
        # restore truncated tool_call ids from the observed-id cache
        ...
    return data
```

This was the most technically satisfying thing I built, and it was genuinely correct for the problem it targeted. Sequential tool calls started working through the proxy.

Parallel calls still failed, because restoring an ID is not the same as *having* a signature. The second call in a parallel batch never had one to cache. There is nothing to restore.

That is the moment the real shape of the bug became visible — and the moment I understood it was not a truncation bug at all.

## Attempt 3: sequentialize the parallel turn

Obvious next move: if the backend wants one call per turn, rewrite history so each parallel call becomes its own assistant turn, each with its own tool result. Perfectly balanced, one call and one response per turn.

From that session:

> the error changed from response-count mismatch to **one replayed bash call missing `thought_signature`**

Progress, of a sort. A new error is information. This one said: you cannot manufacture a Gemini model turn. Splitting one real turn into two synthetic ones means the second synthetic turn contains a function call with no signature, and an unsigned call in its own turn is worse than an unsigned call riding alongside a signed one.

**The signature belongs to the turn, not to the call.** Any transform that creates a new turn creates a turn that cannot be signed.

That single sentence is the whole bug. It took weeks to be able to write it.

## Attempt 4: patch the client

At this point the tooling itself was suspect, so I forked the agent, added logging around message conversion, and confirmed the client was mangling tool call IDs. That turned into an upstream issue and fix — [oh-my-pi#8641](https://github.com/can1357/oh-my-pi/issues/8641), fixed by [#8642](https://github.com/can1357/oh-my-pi/pull/8642) — which preserves the complete opaque ID and replays it byte-for-byte in both the assistant `tool_calls[].id` and the tool result `tool_call_id`.

That fix is real and it shipped. Sequential multi-turn tool calls through custom gateways work because of it.

It also explicitly scoped out the remaining problem:

> A separate parallel-tool-result grouping problem was observed at the gateway layer and is intentionally out of scope for this issue.

Which is where things sat. Sequential worked. Parallel 400'd. Using a coding agent while praying it does not batch two file reads is not a workflow.

Along the way I also ran the same model through a second, unrelated agent ([OpenCode](https://github.com/sst/opencode)) purely as a control. Same gateway, same model, same class of failure. That was worth the hour: it moved the suspect from "my client" to "the translation layer", and stopped me from patching a client that was already correct.

---

## What actually worked

Stop trying to make the model behave. Fix the *history you replay*.

If the backend can only attribute one signed call per turn, then send exactly one call per turn — but do not fabricate turns, and do not throw away work. Keep the signed call. Drop the unsigned siblings from the replayed assistant message. Merge their results into the surviving tool result.

```text
what the agent executed          what gets replayed
─────────────────────────        ──────────────────────────
assistant                        assistant
├── read a.json   [signed]       └── read a.json   [signed]
└── read b.json   [bare]
                                 tool (for the signed call)
tool ← result a                  ├── result a
tool ← result b                  └── [parallel call read(b.json)]
                                     result b
```

Both tools still ran. Both results still reach the model. The wire shape is one function call part and one function response part, which is the only shape the translation layer can produce.

The agent I use ([Oh My Pi](https://github.com/can1357/oh-my-pi)) exposes a `before_provider_request` event where an extension can replace the outgoing payload, so this needs no fork:

```ts
export const SIGNATURE_MARKER = "||thought_signature||";

export function collapseParallelToolCalls<T>(payload: T): T {
	if (typeof payload !== "object" || payload === null || !("messages" in payload)) return payload;
	const raw = payload.messages;
	if (!Array.isArray(raw)) return payload;
	const messages: Message[] = raw;

	// Only engage for the gateway shape this works around: a signature-bearing
	// tool call id. Other providers replay parallel calls correctly and must not
	// have their history rewritten.
	const affected = messages.some(
		(message) =>
			message?.role === "assistant" &&
			Array.isArray(message.tool_calls) &&
			message.tool_calls.length > 1 &&
			message.tool_calls.some((call) => call?.id?.includes(SIGNATURE_MARKER)),
	);
	if (!affected) return payload;

	// ... keep the signed call, fold sibling results into its tool message
}

export default function geminiSerialTools(pi: ExtensionApi): void {
	pi.on("before_provider_request", (event) => collapseParallelToolCalls(event.payload));
}
```

The guard matters more than the transform. No signature marker, no rewrite — so Claude and GPT keep native parallel tool calls, untouched. A compatibility shim that quietly degrades every other provider is not a fix, it is a tax.

Verified live against the gateway:

| Scenario | Before | After |
|---|---|---|
| Single completion | works | works |
| Sequential multi-turn tool calls | works (needs #8642) | works |
| 2 parallel calls, same tool | HTTP 400 | both results returned |
| 3 parallel calls, two different tools | HTTP 400 | all three results returned |
| Another tool round after a collapsed turn | n/a | works |
| Non-Gemini providers | native parallel | native parallel, untouched |

---

## The debugging lessons cost more than the fix

**A stale environment variable can invalidate every test you run.**

Late in the day I had the extension working in an isolated test profile and failing in my real configuration. I bisected the extension list. I inspected the settings database. I read the merge-precedence docs. I built two hypotheses about deduplication and disproved both.

The actual cause: my shell inherited `PI_CODING_AGENT_DIR` pinned to a sandbox profile. Every "real profile" run was reading a completely different config directory. My edits to the real config were never loaded, by anything, ever.

The tell was there the whole time — the model greeted me by name in a directory with no context file, which meant it was reading *someone's* config, just not the one I was editing. I noticed the anomaly and did not chase it. Chase the anomaly.

**Order matters when your probe and your fix hook the same event.**

I wrote a tiny probe extension that logged tool call counts, saw the untransformed payload, and concluded my fix had not run. Wrong: handlers run in registration order, and command-line extensions register *before* config-listed ones. My probe was reporting the state before my own transform. A probe that runs first cannot observe a fix that runs later — put the probe last, or you will diagnose the opposite of reality.

**Make the reproduction cheap before you iterate on it.**

My first dozen reproduction runs shipped a large context file, the full skill and rule set, every MCP tool definition, and multi-kilobyte tool results — per attempt. The bug needs none of that. It needs two tool calls in one turn.

The harness that replaced it: a bare directory, three one-line JSON files, and a minimal agent config listing exactly one extension.

```text
gsd-lab/
├── a.json          {"alpha": 1}
├── b.json          {"beta": 2}
├── c.json          {"gamma": 3}
└── agentdir/
    ├── config.yml  one extension, nothing else
    └── models.yml  one model
```

Runs dropped from expensive and slow to trivial, which is what made the bisecting affordable. If you are going to run something twenty times, spend the first ten minutes making it small. I spent those ten minutes on attempt number fifteen.

**Reproduce in a second client before patching the first.**

The OpenCode control run was the cheapest hour in this entire saga.

---

## Being honest about what this is

This is a workaround, and the post-mortem should say so plainly.

- The model still emits parallel calls; they still execute in parallel locally. Only the replayed history is collapsed.
- The model sees one merged result block instead of N distinct tool results. No data is lost, but the fidelity is lower.
- The unsigned sibling call disappears from history. Its output remains, labeled, inside the surviving result.

A first-class fix belongs one layer down: either the client enforces serialization when a provider is known not to support parallel replay, or provider metadata describes how to group tool results for backends that enforce call/response part parity. I filed that as [oh-my-pi#9167](https://github.com/can1357/oh-my-pi/issues/9167), deliberately written without gateway-specific or employer-specific detail, since the pattern applies to any OpenAI-compatible bridge in front of Gemini.

---

## The takeaway

Four attempts failed because each one accepted the error message's framing. "Response parts must equal call parts" sounds like a counting bug, so I kept fixing counting: restore the IDs, balance the turns, disable parallelism at the proxy. The counts were never wrong on my side.

The real constraint was structural, and one sentence long: **a signature belongs to a turn, and you cannot manufacture a turn.** Once that is the model in your head, the fix stops being clever. Keep the turn the model actually produced, keep its signed call, and make everything else fit inside it.

Two months, and the working diff is smaller than the config file of my first attempt. That ratio is not unusual for protocol bugs. The time goes into building an accurate mental model of a system you cannot see, using error messages written from the far side of a translation layer.

Then you delete the proxy, the callback, the fork, and the four clever workarounds, and you write the thirty lines that were always the answer.
