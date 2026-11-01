---
layout: post
title: "I Gave My Agent a Voice"
date: 2026-11-01
categories: ai agents macos
series: "AI coding agent productivity"
---

macOS `say -v Samantha` is fine for one-liners. Play a full paragraph and the flat stress pattern gives it away in seconds. I wanted narration I could listen to, running fully local on the Mac.

The answer was Kokoro-82M — and it was already sitting inside my agent harness, one command away.

---

## The A/B test

Same sentence, two engines, played back to back:

```bash
# Apple built-in (Samantha, 22kHz AIFF)
say -v Samantha "Hello Ankit. This is the built in Apple voice." -o apple.aiff

# Kokoro-82M q8 via OMP (24kHz WAV, voice af_heart)
omp say "Hello Ankit. This is Kokoro, the new local voice." \
  --voice af_heart --out kokoro.wav
```

| | Apple `say` (Samantha) | Kokoro-82M (`af_heart`) |
|---|---|---|
| Engine | Concatenative/formant-era pipeline | 82M-param neural TTS |
| Rhythm | Flat, even stress on every word | Phrase-level stress and pacing |
| Output | 22 kHz AIFF | 24 kHz PCM WAV |
| Verdict | Mechanical | Human enough to narrate with |

Not subtle. Kokoro wins on first listen.

## The install that wasn't

I expected a Homebrew formula, a venv, a README. Instead:

```bash
omp setup speech --check
# [ok] Text-to-Speech model: kokoro
```

OMP's `local` catalog already ships Kokoro as the `speech` role: `onnx-community/Kokoro-82M-v1.0-ONNX`, int8-quantized, ~92 MB. The "quantised model" question I walked in with answered itself — q8 is the default and only build. It matches the pattern on the dictation side, where Handy already runs parakeet `Q8_0` (698 MB) for live transcription.

One real failure on the way there, worth recording: my shell tooling caps commands at 30 seconds. The first `omp say` pulled ~34 MB then got killed, leaving a truncated `model_quantized.onnx` that failed on next run with `Protobuf parsing failed`. Fix was `rm` the truncated file and relaunch detached with `nohup ... &` so the ~90 MB pull could finish. Lesson captured in my agent's gotchas file: never retry a large model pull inside a short timeout window — clear the partial and go detached.

## The 700 MB question

I told my agent I was fine going up to 700–800 MB if quality demanded it. The research says: don't bother, for preset narration.

| Model | Size | What it buys you | Worth it? |
|---|---|---|---|
| Kokoro-82M | ~92 MB q8 | 54 preset voices, Apache 2.0, realtime on CPU | Yes — this post |
| Chatterbox Turbo | 350M params, 4–8 GB VRAM | Voice cloning from a 5–10 s clip; beat ElevenLabs in blind tests | Only if you need *your* voice |
| Qwen3-TTS | ~0.5–1 GB+ | Multilingual streaming, voice design | Only for non-English/cloning |
| Fish Audio S2 Pro | Large, license-limited | Top benchmark scores | No — license + size |

The short answer from every comparison I read: Kokoro for preset narration, Chatterbox for cloning, Qwen3 for multilingual, Fish for benchmarks-you-pay-for. My job is narration. Kokoro at 92 MB ends the search.

## What "later" looks like

Two threads I deliberately left open:

1. **Voice cloning** — Chatterbox Turbo clones from seconds of audio. If I ever want the agent to sound like *me* reading release notes, that's the trial. Cost: a separate install outside OMP, real VRAM.
2. **OMP dictation overlap** — OMP also ships Parakeet (`parakeet-tdt-0.6b-v3`, ~680 MB) for its `dictation` role. I don't need it: Handy already owns the mic with its own parakeet build. Redundant downloads are how you end up with three copies of the same weights — my Oct 22 Whisper-port post was exactly that lesson.

## The one-line setup

```bash
omp say "anything" --voice af_heart --out out.wav
```

Twelve voices (`af_heart`, `af_bella`, `am_michael`, `bf_emma`, …), speed 0.5–2.5, fully offline. The mechanical voice is gone.

---

## Source

- Kokoro repo: [hexgrad/kokoro](https://github.com/hexgrad/kokoro) · weights: [hexgrad/Kokoro-82M](https://huggingface.co/hexgrad/Kokoro-82M) · ONNX q8: [onnx-community/Kokoro-82M-v1.0-ONNX](https://huggingface.co/onnx-community/Kokoro-82M-v1.0-ONNX)
- Replaced-macos-say walkthrough: [lukasrieder.com](https://www.lukasrieder.com/blog/i-replaced-macos-say-with-local-neural-speech/)
- Best local TTS on Mac 2026: [murmurtts.com](https://www.murmurtts.com/blog/best-local-tts-models-2026)
- Kokoro vs Chatterbox vs Qwen3 vs Fish head-to-head: [tryspeakeasy.io](https://www.tryspeakeasy.io/blog/open-source-text-to-speech-2026)
- Cloning path: [ResembleAI/chatterbox-turbo-ONNX](https://huggingface.co/ResembleAI/chatterbox-turbo-ONNX) · [resemble-ai/chatterbox](https://github.com/resemble-ai/chatterbox)
- Multilingual path: [QwenLM/Qwen3-TTS](https://github.com/QwenLM/Qwen3-TTS) · benchmark path: [fishaudio/fish-speech](https://github.com/fishaudio/fish-speech)
- My earlier weight-duplication lesson: {% post_url 2026-10-22-eight-hundred-mb-three-404s-deleted-download %}
