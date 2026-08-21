---
layout: post
title: "Format Contamination: Why Your AI Agent Stopped Calling Tools"
date: 2026-08-21 15:35:00 +0530
categories: ai agents productivity
series: "AI coding agent productivity"
---

You hand off an active terminal session from one AI model to another—say, from Claude to Gemini. The history transfers perfectly. But the moment the new model takes over, it stops emitting native JSON tool calls. Instead, it just prints text like `[called bash({"command": "ls"})]` directly into the chat. It's broken, and it's hallucinating a tool-calling syntax that doesn't exist.

Why did the model suddenly forget how to use tools? Because it read its own history, and we accidentally taught it the wrong lesson.

Here is the story of how we broke our agent, the debugging rabbit hole that followed, and the architectural fix provided by the upstream maintainers.

### The Problem: API Rejection on Handoff

In multi-model agent frameworks like Oh-My-Pi (OMP), moving a conversation between models requires "flattening" the previous model's tool calls. If Claude made a tool call using its specific native signature, Gemini's API will reject that foreign signature as invalid when it inherits the context.

To fix this, we wrote an extension to intercept the outgoing payload right before the network request (`before_provider_request`). We converted the old foreign tool calls into flat text strings, preserving the context without crashing the gateway. 

Our flattening logic looked like this:

```typescript
// BAD: Formatting a flattened tool call as the assistant's own voice
parts.push(`[called ${name}(${args})]`);
```

This successfully bypassed the API validation errors. But it created a semantic poison pill.

### The Trap: Imitating the Assistant's Voice

From the new model's perspective, it opened its eyes, looked at the `role: "assistant"` messages in its context window, and saw that its "past self" executed tools by just writing `[called bash(...)]` in plain text. 

LLMs are fundamentally few-shot learners that heavily weight their own conversational history. The model saw a pattern, assumed it was the required output format, and obediently imitated it. Native tool calling silently degraded into prose.

We didn't realize this immediately. At first, we assumed the OMP platform or the API gateway was failing to strip internal signatures properly. We filed an issue upstream ([oh-my-pi#9182](https://github.com/can1357/oh-my-pi/issues/9182)), convinced that the platform's core serialization logic was broken.

### The Upstream Assist: Wire Format vs. Native Provenance

The OMP maintainer, `roboomp`, quickly set us straight. 

First, they clarified that the platform's internal signature stripping only applied to native Google routes, not the OpenAI-compatible gateway route we were using. Our extension *was* necessary, but our implementation was flawed.

More importantly, `roboomp` gave us a massive architectural hint. We were hooking into the serialization layer (`before_provider_request`), trying to guess which turns were foreign by checking if their tool calls lacked a specific `||thought_signature||` marker. 

Heuristics based on the *absence* of a marker are brittle. If the gateway ever changed its ID formats, native calls would be mistakenly flattened.

Instead, `roboomp` suggested we move our intercept hook earlier in the framework lifecycle: the `context` event. At this stage, we have access to native `AgentMessage` objects that carry explicit `model`, `provider`, and `api` fields. 

| Approach | Where it hooks | How it detects foreign turns | Failure Mode |
|---|---|---|---|
| **Wire Format (Old)** | Network Serialization (`before_provider_request`) | Guesses based on *missing* proprietary signature markers in tool call IDs. | If the gateway changes ID formats, native calls are mistakenly flattened. |
| **Native Provenance (New)** | Framework Context (`context`) | Explicit comparison of `message.model` vs current active model. | Safely isolates foreign turns with 100% accuracy before serialization. |

Provenance is factual, whereas wire-format sniffing is a guess.

### The Fix: Third-Person Records at the Context Layer

Armed with this advice, we refactored our extension. We moved the detection logic to the `context` hook, explicitly comparing `message.model` to the current active model.

Then, we fixed the format contamination itself. The solution isn't to stop flattening history—foreign signatures genuinely cannot be manufactured. The fix is to stop rendering those flattened calls in the assistant's own voice.

If a tool call must be flattened into history, it should be rendered as an inert, third-person record (e.g., inside a `user` or `system` role) that carries the facts without modeling an action the assistant itself performs. 

```typescript
// GOOD: Dropping the imitable call syntax entirely, retaining only the result context in a non-assistant role
userParts.push(`[Result of prior ${name} execution: ${result}]`);
```

By removing the `[called ...]` template from the assistant's history, the model no longer has a bad example to follow, and native tool-calling reliability returns immediately.

### Source
* Testing and discovery performed against Oh-My-Pi (OMP) `v17.3.8`.
* Discussion and architectural guidance from [oh-my-pi#9182](https://github.com/can1357/oh-my-pi/issues/9182).