---
layout: post
title: "I Gave My Agent a Voice"
date: 2026-11-01
categories: ai agents macos
series: "AI coding agent productivity"
---

macOS [`say -v Samantha`](https://developer.apple.com/library/archive/documentation/LanguagesUtilities/Conceptual/MacAutomationScriptingGuide/SpeakText.html) is fine for one-liners. Play a full paragraph and the flat stress pattern gives it away in seconds — every word gets the same weight, like a metronome reading the news. I wanted narration I could actually listen to, running fully local on the Mac, and I walked in assuming it would cost me an afternoon of Homebrew, venvs, and model wrangling.

The answer was [Kokoro-82M](https://github.com/hexgrad/kokoro) — and it was already sitting inside my agent harness, one command away, [int8-quantized](https://huggingface.co/onnx-community/Kokoro-82M-v1.0-ONNX) at ~92 MB.

---

## Why Apple [`say`](https://developer.apple.com/library/archive/documentation/LanguagesUtilities/Conceptual/MacAutomationScriptingGuide/SpeakText.html) sounds mechanical

Apple's built-in voices are a concatenative/formant-era pipeline: small recorded fragments stitched together with rule-based prosody. For short alerts it's fine. But the stitching shows on anything longer than a sentence — uniform syllable stress, no phrase-level pacing, 22 kHz ceiling. [Lukas Rieder's writeup](https://www.lukasrieder.com/blog/i-replaced-macos-say-with-local-neural-speech/) on replacing `say` with local neural speech makes the same point: a modern 82M-param neural model does in realtime on CPU what used to need a server farm, and the difference is phrase-level rhythm, not just timbre.

Kokoro is exactly that model: 82M parameters, 54 preset voices, [Apache 2.0](https://huggingface.co/hexgrad/Kokoro-82M), realtime on a laptop CPU, 24 kHz output. It doesn't clone voices — it ships good ones.

## The A/B test

Same idea, two engines, played back to back on the MacBook speakers:

```bash
# Apple built-in (Samantha, 22kHz AIFF)
say -v Samantha "Hello Ankit. This is the built in Apple voice. \
Notice the flat rhythm and even stress on every word." -o apple.aiff

# Kokoro-82M q8 via OMP (24kHz WAV, voice af_heart)
omp say "Hello Ankit. This is Kokoro, the new local voice. \
Notice the natural rhythm and stress." \
  --voice af_heart --out kokoro.wav
```

| | Apple `say` (Samantha) | Kokoro-82M (`af_heart`) |
|---|---|---|
| Engine | Concatenative/formant-era pipeline | 82M-param neural TTS |
| Rhythm | Flat, even stress on every word | Phrase-level stress and pacing |
| Output | 22 kHz AIFF | 24 kHz PCM WAV |
| Verdict | Mechanical | Human enough to narrate with |

Not subtle. Kokoro wins on first listen — the stress lands on the right words instead of all of them.

## The install that wasn't

I expected a Homebrew formula, a venv, a README. Instead:

```bash
omp setup speech --check
# [ok] Text-to-Speech model: kokoro
```

OMP's `local` catalog already ships Kokoro as the `speech` role: [`onnx-community/Kokoro-82M-v1.0-ONNX`](https://huggingface.co/onnx-community/Kokoro-82M-v1.0-ONNX), int8, ~92 MB. The "quantised model" question I walked in with answered itself — q8 is the default and only build, same pattern as the dictation side where Handy already runs parakeet `Q8_0` (698 MB) for live transcription.

Twelve voices ship with it — `af_heart` (default), `af_bella`, `af_nicole`, `af_aoede`, `af_kore`, `af_sarah`, `am_michael`, `am_fenrir`, `am_puck`, `bf_emma`, `bm_george`, `bm_fable`. Switching voices downloads nothing; speed is 0.5–2.5× via `tts.localSpeed`. Fully offline.

One real failure on the way there, worth recording: my shell tooling caps commands at 30 seconds. The first `omp say` pulled ~34 MB of the ~90 MB model, got killed, and left a truncated `model_quantized.onnx` that failed on the next run with `Protobuf parsing failed`. The fix was `rm` the truncated file and relaunch detached with `nohup ... &` so the download could finish past the timeout. Lesson, now in my agent's gotchas file: never retry a large model pull inside a short timeout window — clear the partial file and go detached.

## The 700 MB question

Mid-session I told my agent I was fine going up to 700–800 MB if quality demanded it. The research says: don't bother, for preset narration. Two reviews did the heavy lifting — [Murmur's 14-minute Mac-local roundup](https://www.murmurtts.com/blog/best-local-tts-models-2026) with 12 listenable voice samples, and [SpeakEasy's head-to-head](https://www.tryspeakeasy.io/blog/open-source-text-to-speech-2026) (Kokoro vs Chatterbox vs VibeVoice vs Qwen3-TTS vs Fish, "pick one in 2 minutes"). Their verdicts converge:

| Model | Size | What it buys you | Worth it? |
|---|---|---|---|
| [Kokoro-82M](https://huggingface.co/hexgrad/Kokoro-82M) | 82M params, ~92 MB q8, 2–3 GB VRAM | 54 preset voices, Apache 2.0, realtime CPU | Yes — this post |
| [Chatterbox Turbo](https://github.com/resemble-ai/chatterbox) ([ONNX](https://huggingface.co/ResembleAI/chatterbox-turbo-ONNX)) | 350M params, 4–8 GB VRAM | Voice cloning from a 5–10 s clip; [beat ElevenLabs in blind tests](https://www.tryspeakeasy.io/blog/open-source-text-to-speech-2026); MIT | Only if you need *your* voice |
| [Qwen3-TTS](https://github.com/QwenLM/Qwen3-TTS) | ~0.5–1 GB+ | Multilingual streaming, voice design | Only for non-English / streaming |
| [Fish Audio S2 Pro](https://github.com/fishaudio/fish-speech) | Large, license-limited | Top benchmark scores | No — license + size |
| VibeVoice | Large | Up to 90 min multi-speaker generation | No — different job |

The short answer from both reviews: **Kokoro if you want fast and cheap, Chatterbox if you need voice cloning in English, VibeVoice for long multi-speaker audio, Qwen3-TTS for multilingual streaming, Fish if benchmark quality is all that matters and you can pay for a license.** My job is narration of agent output. Kokoro at 92 MB ends the search.

Murmur's framing stuck with me: *the best model changes with the job*. Kokoro is "a small preset-voice starting point" — and for narration, the starting point is the whole trip.

## Why not Parakeet too?

OMP's catalog also offers a `dictation` role — [`parakeet-tdt-0.6b-v3`](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3), ~680 MB, sherpa-onnx int8. I skipped it deliberately: [Handy](https://handy.computer/) already owns the mic with its own parakeet build (`parakeet-unified-en-0.6b` Q8_0, 698 MB) for live transcription. Downloading OMP's Parakeet would buy a second copy of the same weights for a mic button I don't press. Redundant downloads are how you end up with three copies of the same model — my [Oct 22 Whisper-port post]({% post_url 2026-10-22-eight-hundred-mb-three-404s-deleted-download %}) was exactly that lesson. One engine per job.

## What "later" looks like

Two threads deliberately left open:

1. **Voice cloning** — [Chatterbox Turbo](https://huggingface.co/ResembleAI/chatterbox-turbo-ONNX) clones from seconds of audio and beat ElevenLabs blind. If I ever want the agent to sound like *me* reading release notes, that's the trial. Cost: a separate install outside OMP, real VRAM (4–8 GB).
2. **The Rust-native path** — [`sayneat`](https://gitlab.com/parlant-co/sayneat) (Rust TTS CLI, GitLab) now ships four local engines with Kokoro recommended: int8 (~147 MB, 103 EN+ZH voices), a full-precision `kokoro-hq` (~365 MB, skips the high-frequency filter that softens int8 hiss), a compact `kitten` (~68 MB, 8 voices), and 44.1 kHz `supertonic` (~129 MB). If the int8 hiss ever bothers me, `kokoro-hq` is the 365 MB answer — still inside my 700 MB budget.

## The one-line setup

```bash
omp say "anything" --voice af_heart --out out.wav
```

Twelve voices, speed 0.5–2.5, fully offline, 92 MB. The mechanical voice is gone — and the afternoon I budgeted for model wrangling went to writing this instead.

---

## Source

- Kokoro repo: [hexgrad/kokoro](https://github.com/hexgrad/kokoro) · weights: [hexgrad/Kokoro-82M](https://huggingface.co/hexgrad/Kokoro-82M) · ONNX q8: [onnx-community/Kokoro-82M-v1.0-ONNX](https://huggingface.co/onnx-community/Kokoro-82M-v1.0-ONNX)
- Replaced-macos-say walkthrough: [lukasrieder.com](https://www.lukasrieder.com/blog/i-replaced-macos-say-with-local-neural-speech/)
- Best local TTS on Mac 2026, 12 voice samples: [murmurtts.com](https://www.murmurtts.com/blog/best-local-tts-models-2026)
- Kokoro vs Chatterbox vs VibeVoice vs Qwen3 vs Fish head-to-head: [tryspeakeasy.io](https://www.tryspeakeasy.io/blog/open-source-text-to-speech-2026)
- Cloning path: [ResembleAI/chatterbox-turbo-ONNX](https://huggingface.co/ResembleAI/chatterbox-turbo-ONNX) · [resemble-ai/chatterbox](https://github.com/resemble-ai/chatterbox)
- Multilingual path: [QwenLM/Qwen3-TTS](https://github.com/QwenLM/Qwen3-TTS) · benchmark path: [fishaudio/fish-speech](https://github.com/fishaudio/fish-speech)
- Parakeet dictation model: [nvidia/parakeet-tdt-0.6b-v3](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3) · mic app: [Handy](https://handy.computer/)
- My earlier weight-duplication lesson: {% post_url 2026-10-22-eight-hundred-mb-three-404s-deleted-download %}
