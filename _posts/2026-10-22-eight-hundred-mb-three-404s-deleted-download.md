---
layout: post
title: "An 800 MB Model File, Three 404s, and One Deleted Download: Porting Whisper Dictation to macOS"
date: 2026-10-22 06:10:00 +0530
categories: macos ai productivity
series: "Moving to Linux"
---

Apple's built-in dictation garbled every technical sentence I spoke, so I replaced it with a local Whisper model on my Mac. It took three failed downloads, two wrong model keys, and one deleted file to get an 800 MB download right.

The punchline is a two-line fix. The journey is why I'm writing it down.

## The sealed engine

macOS dictation runs as DictationIM → corespeechd → localspeechrecognition. There is no model slot. You cannot swap in Whisper, point it at a GGUF file, or tune a parameter. Apple ships its own on-device recognizer, and the knobs exposed to you are: language pack, microphone level, hotkey. I verified all three — correct pack (`en_IN`), mic at 100, OS current — and three stale recognizer processes that survive even `kill -9`. The engine is what it is: a general-purpose keyboard replacement with no domain vocabulary and no correction layer.

So: replace, don't tune.

## The model that was already staged

My Linux workstation already ran [Voxtype](https://github.com/peteonrails/voxtype) with Whisper Medium 8-bit quantized (`ggml-medium.en-q8_0.bin`, 786 MB) — same accuracy as full FP16 at half the memory. The Mac had Voxtype staged but unconfigured, defaulting to `base.en`. The job looked trivial: point it at the quant.

It was not trivial. The failure cascade:

1. **Wrong default.** I set `medium.en` (full ~1.5 GB) instead of the quant. Caught by review, not by tooling.
2. **Wrong catalog key.** I set `medium.en-int8` — but that's an openvino-engine key, and the whisper engine's downloader only knows `ggml-*.bin` names. `voxtype setup --download` glued the key onto the Hugging Face base URL and returned 404 twice.
3. **Deleted 785 MB of good bytes.** An earlier `curl -C -` resume had left the file 1 MB short of upstream's 823,382,461 bytes. Instead of resuming the last megabyte against the resolved CDN URL, I deleted the file and started over. Twice.

## The two-line fix

Resolve the real CDN target first (HF `/resolve/` URLs serve a 302 stub — the earlier resume failed because it resumed against the stub, not the target), then download against it with auth:

```bash
RURL=$(curl -sIL -o /dev/null -w '%{http\_code}' --max-time 25 \
  https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-medium.en-q8_0.bin)
export HF\_TOKEN=$(cat ~/.cache/huggingface/token)
curl -L --fail --retry 5 -C - -H "Authorization: Bearer $HF\_TOKEN" \
  -o ~/.local/share/voxtype/models/ggml-medium.en-q8_0.bin "$RURL"
```

Two things made the difference: the resolved URL (authed requests skip anonymous throttling — throughput went from ~1 MB/s to ~3 MB/s) and `-C -`, which makes every retry a resume instead of a restart.

Config, for the record:

```toml
[whisper]
model = "ggml-medium.en-q8_0.bin"
initial_prompt = "Voxtype, Jeeves, OMP, Pensando, AMD, DPU, RDMA, SmartNIC"
```

## What I checked before giving up on Apple

Not nothing. Web search plus local audit turned up the full option space:

| Option | Verdict |
|---|---|
| Voice Control custom vocabulary (up to 1000 terms) | Real fix for known jargon, useless for new mishearings |
| "Improve Siri & Dictation" toggle | Sends clips to Apple for grading — improves their model, not yours |
| Apple Intelligence / Siri switches | No vocabulary or training lever for the dictation recognizer |
| Kill hung `corespeechd`, re-add language pack | Worth doing once; orphans recurred here anyway |
| [Handy](https://github.com/cjpais/Handy) (free, MIT, on-device) | Same Whisper accuracy, worse Mac integration; watch [PR #1680](https://github.com/cjpais/Handy/pull/1680) for Apple's SpeechAnalyzer as a zero-download model |
| [Superwhisper](https://superwhisper.com) ($85/yr) | The paid answer: per-app modes + LLM cleanup pass |

None of them move Apple's accuracy ceiling. The quant does.

## Lessons

- **Resume, never re-download.** `curl -C -` against the resolved CDN URL fetches only the missing tail. Deleting a 99.9% complete file is an environmental crime.
- **Match the app's catalog, not upstream filenames.** A valid HF filename is not a valid downloader key. Use the app's own `setup --download --model <key>` or bypass the downloader entirely — never a hybrid of both.
- **Resolve redirects before reasoning about bytes.** A 1 MB "shortfall" against a 302 stub is not a shortfall. `curl -sIL` + `content-length` first, conclusions after.

Total damage: ~2.4 GB downloaded for one 786 MB file. The dictation works now.

## Source

- [Voxtype](https://github.com/peteonrails/voxtype) — headless speech-to-text daemon
- [whisper.cpp](https://github.com/ggerganov/whisper.cpp) — pre-quantized Whisper models at [huggingface.co/ggerganov/whisper.cpp](https://huggingface.co/ggerganov/whisper.cpp)
- [Handy](https://github.com/cjpais/Handy) — free MIT on-device dictation; [Apple Speech PR #1680](https://github.com/cjpais/Handy/pull/1680)
- [Superwhisper](https://superwhisper.com) — commercial Mac dictation with cleanup modes
