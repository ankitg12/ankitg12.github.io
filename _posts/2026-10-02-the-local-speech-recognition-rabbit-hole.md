---
layout: post
title: "The Local Speech Recognition Rabbit Hole: Wayland, Vulkan, and the Streaming Illusion"
date: 2026-10-02 21:45:00 +0530
categories: linux ai productivity
series: "Moving to Linux"
---

All I wanted was push-to-talk voice dictation on my Linux developer workstation.

On Windows, pressing `Win+H` fires up a serviceable cloud dictation widget. After migrating my daily workflow to an AMD Ryzen AI workstation running Ubuntu 24.04 and GNOME Wayland, I wanted something completely local, private, and zero-latency: tap a physical key on my keyboard, speak a sentence, and have the words cleanly typed at my cursor—whether inside a WezTerm terminal, a browser text area, or an IDE.

What looked like an afternoon utility setup spiraled into a deep dive across Linux input subsystems (`evdev`, `uinput`, `ydotool`), SoundWire array microphone clipping, Vulkan compute on integrated GPUs, the memory arithmetic of 8-bit quantization, and how easy it is to confuse the different architectural meanings of "streaming" in speech recognition.

Here is the full story, the benchmarks, and the architectural lessons learned from breaking voice dictation five different ways in a single day.

---

## Act I: The Handy Honeymoon (and the Wayland Reality Check)

When setting up voice input on my new Linux workstation, my initial choice was straightforward: install [Handy](https://github.com/cjpais/Handy). 

I had previously installed and tested Handy on Windows, where its clean Tauri GUI and local model management worked reliably. When I moved to Linux, reaching for Handy felt like natural continuity—it offers a pre-packaged AppImage, bundled local models, and cross-platform familiarity.

On paper, Handy's architecture is brilliant:
- Built around `whisper.cpp` and `parakeet.cpp`.
- Pulls compact, 8-bit quantized GGUF models directly from Hugging Face (`whisper-medium-Q8_0.gguf`, 794 MB).
- Features a clean UI for switching engines and testing microphone inputs.

I installed Handy v0.9.7, wired it to a global GNOME shortcut (`Ctrl+Super+H`), and gave it a spin. Within twenty minutes, the Linux desktop reality check arrived.

### 1. The SoundWire VAD Failure
On my laptop, the internal microphone array operates over the Intel/AMD SoundWire digital audio bus. When dictating short phrases through Handy, the transcripts consistently came back completely empty. 

Digging into the logs, Handy's bundled Silero Voice Activity Detector (VAD) was aggressively misclassifying short SoundWire audio frames as silence and dropping them before they ever reached the transcription model. Disabling VAD helped, but replaying the captured WAV buffers through PipeWire revealed timing glitches where Handy's audio pipeline clipped buffer boundaries (which I filed upstream as Handy issue [#2187](https://github.com/cjpais/Handy/issues/2187)).

### 2. Wayland Hotkey and Typing Friction
To type text into the active window, Handy relied on simulated `Ctrl+V` clipboard pastes via `enigo`. On Wayland, this caused constant friction:
- Every dictation clobbered whatever code snippet was in my system clipboard.
- The AppImage GUI occasionally stole window focus, preventing typing into the terminal underneath.
- Global desktop shortcuts (`Ctrl+Super+H`) felt clumsy. I wanted a **single dedicated physical key**—specifically `Numpad 0/Ins` (`KEY_KP0`)—that acted as an ergonomic push-to-talk toggle across every application without desktop environment interference.

I didn't want a cross-platform Tauri GUI app with a 1 GB WebKit memory footprint sitting in my tray. I needed something deeply Linux-oriented: a headless Unix daemon built around native kernel interfaces.

---

## Act II: Entering Voxtype and the "In Box Type" Defect

That search led me to [Voxtype](https://github.com/peteonrails/voxtype): a headless speech recognition daemon written in Rust, designed from the ground up specifically for Linux, Wayland, and Sway.

Voxtype solves the desktop integration problem the Unix way:
- **Direct Kernel `evdev` Monitoring:** It grabs `/dev/input/event*` devices directly. It doesn't care whether you're in GNOME, Sway, a TTY, or a lock screen—it captures physical scancode 82 (`EVTEST_82`, Numpad 0) with zero desktop shortcut overhead.
- **Native Wayland Typing:** It routes text through `ydotool` and `/dev/uinput`, emitting genuine synthetic key events directly into the compositor without touching the clipboard.
- **Lightweight Systemd Daemon:** Runs as `systemctl --user start voxtype`, consuming just 200 MB of RAM.

I configured Voxtype with `mode = "toggle"` on Numpad 0, enabled a shell post-processor to wrap dictated text in `[v] ... [/v]` tags, and tested it in my terminal.

I tapped Numpad 0, said *"which transcription model we are currently using in voxtype"*, and hit the key again.

At my cursor, Voxtype typed:
```text
[v] which transcription model we are currently using in box type [/v]
```

*"in box type"*.

I opened `~/.config/voxtype/config.toml` and checked the default engine:
```toml
engine = "whisper"
[whisper]
model = "base.en"
```

Voxtype had defaulted to OpenAI Whisper's **`base.en`**—a tiny 74-million-parameter model running on CPU threads. While it loaded quickly, its acoustic modeling capacity was so primitive that common technical terms degraded into phonetic gibberish.

In Handy, dictation had felt noticeably sharper. That sparked the rabbit hole: *Why not switch to Parakeet, or put Whisper on the local GPU?*

---

## Act III: Whisper vs. Parakeet (and the UMA GPU Surprise)

My workstation is powered by an AMD Ryzen AI 7 PRO 350 processor featuring Zen 5 CPU cores and an integrated AMD Radeon 860M GPU (`GFX1152`). 

The standard mental model in AI is simple: *"GPUs are always faster than CPUs."* 

To verify this, I downloaded both engines and benchmarked them against the exact same real-world voice recording (`11.91 seconds` of 16 kHz mono speech):

| Engine | Model & Precision | Execution Hardware | Latency | Real-Time Factor | Transcription Accuracy |
| :--- | :--- | :--- | :---: | :---: | :--- |
| **Parakeet-TDT** | 0.6B FastConformer (int8) | **CPU (Zen 5 AVX-512)** | **0.33 s** | **36.1x** | *"re-downloaded models"* (Exact) |
| **Whisper Base** | Base.en (fp16) | **GPU (Radeon 860M Vulkan)** | 0.68 s | 17.5x | *"re-downloader models"* (Phonetic error) |
| **Whisper Base** | Base.en (fp16) | **CPU (8 threads)** | 0.86 s | 13.8x | *"re-downloader models"* |
| **Whisper Medium** | Medium.en (Q8_0) | **GPU (Radeon 860M Vulkan)** | 1.43 s | 8.3x | *"pre downloaded models"* |

The numbers completely contradicted the naive *"always use GPU"* assumption:
- **Parakeet-TDT 0.6B int8 on CPU** took **0.33 seconds**—processing 12 seconds of speech in a third of a second (36x faster than real-time).
- **Whisper Base on Vulkan GPU** took **0.68 seconds**—twice as slow as Parakeet on CPU.
- **Whisper Medium on Vulkan GPU** took **1.43 seconds**—over 4x slower.

### Why CPU Beat GPU for Push-to-Talk Dictation
The explanation lies in hardware microarchitecture and memory topology:

1. **Unified Memory Architecture (UMA):** An integrated APU does not have dedicated GDDR6 or HBM VRAM with 1,000 GB/s of bandwidth. The CPU cores and the GPU compute units share the same 128-bit LPDDR5 system memory bus (~100 GB/s).
2. **Batch Size = 1:** In push-to-talk dictation, you are transcribing a single short utterance. There is no batch dimension ($N=1$). Matrix-vector multiplications are strictly memory-bandwidth bound, not compute-bound.
3. **The Kernel Launch Tax:** For small models, the overhead of recording Vulkan command buffers, dispatching shaders through the DRM ring buffer, and waiting on completion fences across the CPU-GPU coherence boundary eats up the GPU's compute advantage.
4. **Zen 5 AVX-512 VNNI:** AMD's Zen 5 cores feature native 512-bit vector datapaths. Running 8-bit quantized integer tensors (`int8`), the CPU executes `VPDPBUSD` (Vector Neural Network Instructions) directly inside its 32 MB L3 cache at 4.0+ GHz. It executes trillions of integer operations per second with zero PCIe or driver dispatch overhead.

---

## Act IV: The Quantization Trap (793 MB vs. 1.5 GB vs. 2.4 GB)

While Parakeet was blazingly fast, it had a glaring flaw in Voxtype 1.1.0: it lacked **vocabulary steering**. When I dictated *"whisper or parakeet"*, Parakeet transcribed it phonetically as *"this or current eight"*. 

Whisper, by contrast, supports an `initial_prompt` parameter. By seeding the prompt with domain keywords:
```toml
[whisper]
initial_prompt = "Voxtype, Jeeves, Whisper, Parakeet, Pensando, AMD, DPU, RDMA, SmartNIC, Wayland"
```
Whisper's cross-attention is conditioned on the prompt prefix, ensuring proper nouns and acronyms are recognized flawlessly.

So I decided to run Whisper Medium on the GPU. I triggered Voxtype's automated download:
```bash
voxtype setup --download --model medium.en
```

I watched the download progress bar climb: `500 MB... 1.0 GB... 1.5 GB`.

Wait. In Handy, Whisper Medium had only taken **793 MB**. Why was Voxtype downloading 1.5 GB?

### The Arithmetic of Model Quantization
Whisper Medium contains **769 million parameters** across 24 encoder layers and 24 decoder layers:

- **Unquantized FP16 (Voxtype's Default):** Standard `ggml-medium.en.bin` stores weights as 16-bit half-precision floats (2 bytes per parameter):
  $$769 \times 10^6 \times 2\text{ bytes} \approx 1.538\text{ GB}$$
- **8-Bit Quantization (`Q8_0`, Handy's Default):** Handy downloads pre-quantized GGUF models (`whisper-medium-Q8_0.gguf`). Quantizing weights to 8-bit signed integers (1 byte per parameter) cuts the footprint in half:
  $$769 \times 10^6 \times 1\text{ byte} \approx 769\text{ MB (785 MiB / 793 MB on disk)}$$

`Q8_0` preserves over 99.8% of the FP16 word accuracy while cutting memory bandwidth demands across the shared LPDDR5 bus by 50%. 

Voxtype's automated setup script had simply defaulted to the unquantized FP16 upstream binary. The fix was downloading the official `ggml-medium.en-q8_0.bin` (786 MB) directly from Hugging Face and symlinking it into place. GPU resident memory dropped from 1.53 GB to 822 MB, and latency improved from 1.66s to 1.43s.

---

## Act V: The Streaming Illusion and the Buffer Starvation Cascade

With Whisper Medium Q8_0 running on the GPU, I asked the obvious next question:
*"Can Voxtype show me words live at my cursor while I speak, like Windows dictation does?"*

I opened `config.toml`, enabled sliding-window streaming, and restarted the daemon:
```toml
[whisper]
streaming = true

[streaming]
interval_secs = 0.5   # Re-transcribe every 500ms
type_partials = true  # Type words live at the cursor
```

I tapped Numpad 0 and spoke a test sentence:
*"Okay, let's see if we see the result stream here now."*

At my cursor, text appeared in violent fits and starts:
```text
Okay, let's see if we... see the result. stream. here. now. Seems like really bad quality.
```

Words were dropped, random periods were inserted between syllables, and latency lagged several seconds behind my voice. 

I checked `journalctl --user -u voxtype`:
```text
Oct 02 21:31:35 voxtype[694770]: INFO Transcription completed in 1.17s
Oct 02 21:31:35 voxtype[694770]: WARN [sliding] inference (1.18s) exceeded the tick interval (0.50s) — audio chunks may be getting dropped; buffer at 5.6s
Oct 02 21:31:36 voxtype[694770]: INFO Transcription completed in 1.17s
Oct 02 21:31:36 voxtype[694770]: WARN [sliding] inference (1.18s) exceeded the tick interval (0.50s) — audio chunks may be getting dropped; buffer at 5.7s
```

The mathematical failure mode was staring me in the face:
- Whisper Medium on the GPU took **1.18 seconds** to compute a forward pass.
- I had instructed the runtime to re-transcribe the audio every **0.50 seconds**.
- Because $1.18\text{s} > 0.50\text{s}$, the GPU fell permanently behind. The incoming audio queue starved, the sliding window dropped frames, and the unstable acoustic boundaries caused the decoder to emit broken punctuation (`"stream. here. now."`).

This prompted a deeper realization: **Whisper is fundamentally NOT a streaming model.**

---

## The Three Paradigms of Speech Recognition "Streaming"

In desktop dictation tools, the label *"Streaming"* is often applied to very different architectural approaches that behave quite differently under the hood. In practice, speech recognition architectures fall into three distinct paradigms:

| Paradigm | Exemplar | Acoustic Context | Computational Complexity | Real-Time Streaming Viability |
| :--- | :--- | :--- | :---: | :--- |
| **1. Sliding-Window Pseudo-Streaming** | OpenAI Whisper | Full expanding buffer $[0 \dots t_k]$ re-evaluated every tick | **$O(T^2)$ quadratic explosion** | **Unusable:** Buffer starvation causes dropped audio and stutter. |
| **2. Offline Transducer** | Parakeet-TDT v3 | Full utterance bidirectional spectrogram | **$O(T)$ linear** | **Batch-only:** Monolithic duration skips cannot predict future audio. |
| **3. Cache-Aware Streaming Transducer** | Parakeet Unified 0.6B, Nemotron 3.5 | Chunked causal frames + persisted state cache | **$O(T)$ linear ($O(1)$ per chunk)** | **True real-time:** Zero re-computation of past frames. |

```
1. Sliding-Window Pseudo-Streaming (Whisper)
   Tick 1 (0.5s): [██] -> Whisper -> "Hello"
   Tick 2 (1.0s): [████] -> Whisper -> "Hello world"
   Tick 3 (1.5s): [██████] -> Whisper -> "Hello world this"
   (Re-computes the entire audio history from second 0 on every tick!)

2. Cache-Aware Streaming Transducer (Parakeet Unified)
   Tick 1 (160ms): [██] -> Causal Encoder -> [Cache 1] -> Joint Net -> "Hello"
   Tick 2 (320ms):       [██] + [Cache 1] -> [Cache 2] -> Joint Net -> " world"
   Tick 3 (480ms):             [██] + [Cache 2] -> [Cache 3] -> Joint Net -> " this"
   (Never re-processes past audio; constant O(1) latency per chunk!)
```

1. **Sliding-Window Pseudo-Streaming (Whisper):** Whisper was trained exclusively on 30-second offline audio blocks. To fake streaming, runtimes repeatedly re-run the entire model over an expanding audio buffer $[0, t_k]$ and diff the outputs to find a "stable prefix." Compute complexity scales quadratically ($O(T^2)$). The moment inference time exceeds the tick interval, the pipeline collapses.
2. **Offline Transducers (Parakeet-TDT v3):** While Parakeet-TDT 0.6B is 4x faster than Whisper (0.33s on CPU), standard TDT models are *bidirectional* (the Conformer encoder attends forward and backward across the entire recording). Furthermore, the TDT duration head predicts dynamic skips up to 4 frames into the future. A model cannot skip forward into acoustic frames that have not yet been spoken!
3. **Cache-Aware Streaming Transducers (Parakeet Unified 0.6B):** True streaming requires an architecture trained with *causal convolutions* and an internal *state cache*. As each 160 ms audio chunk arrives, the encoder processes only that chunk, concatenates it with cached activations from prior chunks, and advances. It never re-processes past audio, and latency remains strictly constant regardless of whether you speak for 5 seconds or 50 minutes.

Looking back at Handy's UI, the author had known this all along:
- **Whisper Medium** carried badges for `99 languages` and `Translate`, but **no streaming badge**.
- **Parakeet Unified EN 0.6B** and **Nemotron Streaming 3.5** carried the dedicated **`Streaming`** waveform badge.

---

## Act VI: The 2.4 GB Unquantized ONNX Trap

Naturally, my next thought was: *"Fine, let's switch Voxtype to Parakeet Unified so we can use genuine cache-aware streaming."*

I ran:
```bash
voxtype setup --download --model parakeet-unified-en-0.6b
```

I checked `ls -lh ~/.local/share/voxtype/models/parakeet-unified-en-0.6b/`:
```text
-rw-r--r-- 1 angaur 1.0G Oct  2 21:38 .encoder.onnx.data.part
-rw-r--r-- 1 angaur  42M Oct  2 21:33 encoder.onnx
```

I inspected the HTTP headers:
```text
content-length: 2437091328  (2.43 GB!)
```

Voxtype was downloading a **2.43 GB unquantized FP32 ONNX model**.

In Handy, `parakeet-unified-en-0.6b-Q8_0.gguf` was **698 MB**. Why was Voxtype downloading a 2.43 GB behemoth?

This exposed the final architectural split between the tools:
- **Handy's Clean GGUF Stack:** Built on `parakeet.cpp` and `transcribe.cpp`. Every model in Handy's catalog is pre-quantized to 8-bit integers (`Q8_0`).
- **Voxtype's Clunky ONNX Stack:** Voxtype 1.1.0 uses Microsoft's ONNX Runtime (`onnxruntime`). While it had an int8 quantized build for `parakeet-tdt-0.6b-v3-int8`, whoever packaged `parakeet-unified` dumped the raw float32 ONNX graph straight out of NVIDIA NeMo without quantizing it.

I killed the download immediately. 

And the punchline? Voxtype's own author recognized this exact mess. In Voxtype's developer roadmap (`CLAUDE.md`, lines 360–371):

> *"1.3.0: parakeet.cpp as a ggml/Vulkan Parakeet backend (#483) — 5-6x faster steady-state than ONNX/MIGraphX on AMD, and the only GPU path for AMD and Intel Arc since ORT has no Vulkan EP.* \
> *2.0 (major): Remove ONNX Runtime support entirely, once every ONNX-backed engine has a native ggml/GGUF port..."*

Voxtype's entire forward roadmap is ripping out ONNX and adopting Handy's exact approach: pure GGUF via `parakeet.cpp`.

---

## Summary & Current Architecture

After a full day of profiling, the final daily-driver configuration on my Linux workstation is rock solid:

1. **Daemon:** Headless `voxtype` running as a `systemctl --user` service.
2. **Hotkey:** Physical `Numpad 0/Ins` (`KEY_KP0`, scancode 82) bound directly via Linux `evdev`. Zero desktop shortcut conflicts.
3. **Typing Engine:** Native `ydotool` via `/dev/uinput`. Fast, reliable, and never touches the system clipboard.
4. **Model:** **Whisper Medium Q8_0 (786 MB)** running on the **AMD Radeon 860M GPU via Vulkan** (`ggml-vulkan`).
5. **Mode:** **Batch-on-stop** (toggle on, speak, toggle off). 
   - Decodes a 7-second utterance in **1.19 seconds**.
   - Conditioned with `initial_prompt` to capture technical engineering jargon, acronyms, and proper nouns.
   - Zero hallucination loops, zero dropped frames, and zero buffer starvation.

When Voxtype 1.3 ships with native `parakeet.cpp` GGUF support, I'll switch back to Parakeet for instant sub-300ms transcription. But until then, understanding the physics of your speech recognition stack beats guessing every time.

---

## Sources & References

- [whisper.cpp (Georgi Gerganov)](https://github.com/ggerganov/whisper.cpp) — High-performance C/C++ port of OpenAI Whisper with Vulkan/CUDA backends.
- [Voxtype Daemon (Pete Karl II)](https://github.com/peteonrails/voxtype) — Headless speech-to-text daemon for Linux and Wayland.
- [Handy (C.J. Pais)](https://github.com/cjpais/Handy) — Cross-platform speech-to-text desktop application using GGUF runtimes.
- [NVIDIA NeMo Parakeet-TDT 0.6B](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3) — FastConformer Token-and-Duration Transducer model on Hugging Face.
- [Whisper GGUF / GGML Models](https://huggingface.co/ggerganov/whisper.cpp) — Pre-quantized official Whisper models.
