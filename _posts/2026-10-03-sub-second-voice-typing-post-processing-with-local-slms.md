---
layout: post
title: "Sub-Second Voice Typing Post-Processing with Local SLMs"
date: 2026-10-03 22:20:00 +0530
categories: ai productivity linux
series: "AI coding agent productivity"
---

Voice typing into developer terminals and coding agents frequently falls victim to acoustic misrecognitions: technical acronyms get mangled, domain terms turn into phonetically similar dictionary words, and grammatical homophones like *two* versus *too* lack contextual disambiguation.

Traditional regex and `sed` pipelines do not scale. They require endless manual curation, conflict with one another, and break valid numbers like *two nodes*. Conversely, routing every voice snippet to a cloud LLM gateway introduces 2+ seconds of latency, breaking conversational typing rhythm.

By placing a quantized 1.5B Small Language Model (SLM) running locally on CPU/iGPU immediately behind a local Whisper speech-to-text engine, we achieve context-aware grammatical cleanup in under 350 milliseconds.

## The Architecture

The pipeline divides labor cleanly between acoustic decoding and semantic repair:

1. **Acoustic Front-End (Whisper Medium via Vulkan GPU):** Takes microphone audio and outputs fast, raw phonetics (~200ms).
2. **Post-Processing Pipe (Local SLM):** Feeds raw text into a locally hosted model via standard completions API.
3. **Typing Injector (`ydotool` / clipboard):** Injects cleaned text directly into the focused window.

```
[ Mic Audio ] -> [ Whisper STT ] -> [ Local Qwen 1.5B via llama-server ] -> [ OS Injector ]
                     (~200ms)                    (~100ms)
```

## Why sed Rules Fail

A naive phonetic replacement table breaks legitimate language patterns:

```bash
# Naive sed replacement:
s/\b(two)\b/too/gI

# Input:
"we have two nodes and you two make mistakes"
# Output:
"we have too nodes and you too make mistakes" # Broken!
```

An SLM understands that *two nodes* denotes a cardinal number, while *you two* in the context of an address means adverbial *too*.

## Latency Benchmark: Local SLM vs Gateway

We benchmarked three backends on the same real-world developer dictation:

| Backend | Runtime Model | Latency | Network Dependency |
| :--- | :--- | :--- | :--- |
| **Local `llama-server` (Vulkan/CPU)** | Qwen2.5-1.5B-Instruct Q4_K_M | **0.35s** | Offline (0 KB) |
| **Cloud Fast Model** | GPT-6-Luna | **2.23s** | Required |
| **Cloud Multimodal Gateway** | Gemini 3.8 Flash | **2.74s** | Required |

A 2.5-second pause turns voice dictation into a waiting game. At ~300ms, the response appears almost instantaneously upon releasing the push-to-talk hotkey.

## The Few-Shot Completion Pattern

Instruct models often try to converse when given standard chat templates. To ensure deterministic string transformation with zero conversational drift, use few-shot completions with a strict stop token:

```typescript
const FEW_SHOT_PREFIX = `You are a speech transcription post-processor for a software engineer.
Clean the user speech input. Fix typos, slang, and phonetic speech errors:
- "two" -> "too" (adverbial: "you too", "me too", "too much"). NEVER change numbers ("two nodes").
- "jet" / "jat" -> "chat"
- "task" -> "tsq" (when referring to tasks or task CLI)
- "gitlix" -> "gitleaks"
Output ONLY the corrected text. Never converse. Never add commentary.

Input: mark this done in task right
Output: mark this done in tsq right

Input: we have two nodes and you two make mistakes with jet messages
Output: we have two nodes and you too make mistakes with chat messages

Input: `;
```

## The Bun Post-Processor

Here is the lightweight TypeScript runner wired directly to stdin/stdout:

```typescript
#!/usr/bin/env bun

interface CompletionResponse {
  choices?: Array<{ text?: string }>;
}

const ENDPOINT = "http://127.0.0.1:8082/v1/completions";
const TIMEOUT_MS = 2000;

const input = (await Bun.stdin.text()).trim();
if (!input) process.exit(0);

let result = input;
try {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), TIMEOUT_MS);

  const res = await fetch(ENDPOINT, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model: "qwen2.5-1.5b-instruct",
      prompt: `${FEW_SHOT_PREFIX}${input}\nOutput:`,
      temperature: 0.0,
      max_tokens: Math.max(input.split(/\s+/).length * 2, 40),
      stop: ["\n", "Input:"]
    }),
    signal: controller.signal
  });

  clearTimeout(timeout);

  if (res.ok) {
    const data = (await res.json()) as CompletionResponse;
    const cleaned = data.choices?.[0]?.text?.trim();
    if (cleaned) result = cleaned;
  }
} catch {
  // Fail-safe: fall back to raw input on timeout or failure
}

process.stdout.write(result);
```

## Source

- [llama.cpp Server](https://github.com/ggerganov/llama.cpp)
- [Qwen 2.5 1.5B Instruct GGUF](https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct-GGUF)
