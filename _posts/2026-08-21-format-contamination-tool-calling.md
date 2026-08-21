---
layout: post
title: "Format Contamination: Why Your AI Agent Stopped Calling Tools"
date: 2026-08-21 15:35:00 +0530
categories: ai agents productivity
series: "AI coding agent productivity"
---

You hand off an active terminal session from one AI model to another—say, from Claude to Gemini. The history transfers perfectly. But the moment the new model takes over, it stops emitting native JSON tool calls. Instead, it just prints text like `[called bash({"command": "ls"})]` directly into the chat. It's broken, and it's hallucinating a tool-calling syntax that doesn't exist.

Why did the model suddenly forget how to use tools? Because it read its own history, and we accidentally taught it the wrong lesson.

### The Trap: Imitating the Assistant's Voice

In multi-model agent frameworks, moving a conversation between models requires "flattening" the previous model's tool calls. If Claude made a tool call using its specific native tool-call signature, Gemini's API will reject that foreign signature as invalid.

To fix this, we intercepted the outgoing payload and converted the old foreign tool calls into flat text strings, preserving the context without crashing the gateway. 

Our flattening logic looked like this:

```typescript
// BAD: Formatting a flattened tool call as the assistant's own voice
parts.push(`[called ${name}(${args})]`);
```

This successfully bypassed the API validation errors, but it created a semantic poison pill. From the new model's perspective, it opened its eyes, looked at the `role: "assistant"` messages in its context window, and saw that its "past self" executed tools by just writing `[called bash(...)]` in plain text. 

LLMs are fundamentally few-shot learners that heavily weight their own conversational history. It saw a pattern, assumed it was the required output format, and obediently imitated it. Native tool calling silently degraded into prose.

### The Fix: Third-Person Records

The solution isn't to stop flattening history—foreign signatures genuinely cannot be manufactured. The fix is to stop rendering those flattened calls in the assistant's own voice.

If a tool call must be flattened into history, it should be rendered as an inert, third-person record (e.g., inside a `user` or `system` role) that carries the facts without modeling an action the assistant itself performs. 

```typescript
// GOOD: Dropping the imitable call syntax entirely, retaining only the result context in a non-assistant role
userParts.push(`[Result of prior ${name} execution: ${result}]`);
```

By removing the `[called ...]` template from the assistant's history, the model no longer has a bad example to follow, and native tool-calling reliability returns immediately.

### Wire Format vs. Native Provenance

Fixing the format contamination led to a deeper architectural realization about *where* to flatten history. We were originally trying to do this right before the network request, interacting with the raw OpenAI-compatible JSON wire format.

| Approach | Where it hooks | How it detects foreign turns | Failure Mode |
|---|---|---|---|
| **Wire Format (Old)** | Network Serialization (`before_provider_request`) | Guesses based on *missing* proprietary signature markers in tool call IDs. | If the gateway changes ID formats, native calls are mistakenly flattened. |
| **Native Provenance (New)** | Framework Context (`context`) | Explicit comparison of `message.model` vs current active model. | Safely isolates foreign turns with 100% accuracy before serialization. |

Heuristics based on the *absence* of a marker are brittle. By shifting our intercept hook earlier in the framework lifecycle (the `context` event), we gained access to native `AgentMessage` objects that carry explicit `model`, `provider`, and `api` fields. Provenance is factual, whereas wire-format sniffing is a guess.

### Source
* Testing and discovery performed against Oh-My-Pi (OMP) `v17.3.8`.
* Fix implementation patterns applied to `gemini-serial-tools` extension architecture.