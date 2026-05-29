---
layout: post
title: "Scanning handwritten notes to text on Windows — why Tesseract fails and what to use instead"
date: 2026-05-29
categories: ai tools windows productivity
---

I got a flatbed scanner. My immediate goal: scan handwritten notebook pages and make the text available to my AI coding agent without retyping anything.

Seemed simple. It was not. Here's what I learned.

---

## The setup

- **Scanner**: Canon CanoScan LiDE 400 (bus-powered USB flatbed)
- **OS**: Windows 11 24H2
- **Scanning software**: [NAPS2](https://www.naps2.com/) — free, open source, has a full CLI
- **Goal**: `/scan` slash command in OMP that captures the page and attaches it to the conversation

---

## Problem 1 — the scanner didn't work at all

Plugged it in via a USB hub. Windows detected it (`Get-PnpDevice` showed `Status: OK`). NAPS2 showed the device. Everything looked right.

But every scan attempt returned 0 pages.

Root cause: the LiDE 400 is **bus-powered** — it draws all its power from USB. My hub wasn't delivering enough current. Moving to a direct port fixed it immediately.

If your flatbed scanner detects but doesn't scan: try direct USB first before debugging anything else.

---

## Problem 2 — Windows 11 24H2 broke ESCL USB scanning

The LiDE 400 supports two scan protocols: classic WIA and eSCL (the driverless network/USB protocol). I tried ESCL first since it doesn't require manufacturer software.

It didn't work. After some digging: [Microsoft acknowledged a bug in Windows 11 24H2](https://learn.microsoft.com/en-us/windows/release-health/status-windows-11-24h2#3446msgdesc) where USB scanners using eSCL fail to enumerate correctly. The fix shipped in KB5048667 (December 2024). If you're on a fresh 24H2 install without updates, eSCL-over-USB is broken.

WIA works fine even without the fix. So: use WIA.

---

## The scan command

[NAPS2's CLI](https://www.naps2.com/doc/command-line) handles everything:

```bash
NAPS2 console \
  --driver wia \
  --device "CanoScan LiDE 400" \
  --source glass \
  --dpi 300 \
  --bitdepth color \
  -o scan.jpg \
  -f -v
```

Wrapped this in an OMP extension:

```typescript
const child = spawn(NAPS2, [
  "--driver", "wia",
  "--device", DEVICE,
  "--source", "glass",
  "--dpi", "300",
  "--bitdepth", "color",
  "-o", out,
  "-f", "-v",
], { stdio: "pipe", windowsHide: true });
```

`stdio: "pipe"` is critical — same lesson from the [screenshot extension](/ai/tools/windows/2026/05/27/omp-screenshot-extension.html). Without it the child process inherits OMP's PTY and gets killed.

Result: `/scan` captures the page and attaches `@C:/tmp/scans/scan-<ts>.jpg` to the editor. Scan and ask in one step.

---

## Problem 3 — Tesseract can't read cursive

NAPS2 has built-in OCR via Tesseract:

```bash
NAPS2 console ... --enableocr --ocrlang eng -o scan.pdf
```

The PDF is created. The text layer is garbage:

```
mee
oat af Reh aide
"aaeatd Pg g&
FR sav ® SEBO ey
```

That's from a page of legible (to humans) cursive English. Tesseract is trained on printed text. It's not even close on handwriting.

This is a known, hard limitation. Tesseract doesn't do handwriting.

---

## The fix — vision LLM

Instead of OCR, send the image to a vision-capable LLM. The same Anthropic Claude API that your agent uses for reasoning handles image-to-text natively:

```python
import base64, json, urllib.request

with open(image_path, "rb") as f:
    b64 = base64.b64encode(f.read()).decode()

payload = {
    "model": "claude-sonnet-4-5",
    "max_tokens": 2048,
    "messages": [{
        "role": "user",
        "content": [
            {
                "type": "image",
                "source": {"type": "base64", "media_type": "image/jpeg", "data": b64}
            },
            {
                "type": "text",
                "text": (
                    "Transcribe all handwritten text from this scanned notebook page. "
                    "It may be upside-down or rotated — correct mentally. "
                    "Preserve structure, bullet points, and indentation. "
                    "Mark uncertain words with [?]."
                )
            }
        ]
    }]
}

req = urllib.request.Request(
    "https://api.anthropic.com/v1/messages",
    data=json.dumps(payload).encode(),
    headers={
        "Content-Type": "application/json",
        "x-api-key": os.environ["ANTHROPIC_API_KEY"],
        "anthropic-version": "2023-06-01",
    }
)

with urllib.request.urlopen(req, timeout=30) as resp:
    result = json.loads(resp.read())

print(result["content"][0]["text"])
```

Output on the same page that Tesseract mangled:

```
**Meeting notes — Q3 planning**

- **Action items:**
  - ① Follow up with design team on spec
  - share draft by Friday
  - schedule review [?]

- Blocked on infra
- Need sign-off from Alice
- deadline: end of month

**Budget** → confirm with finance
- carry over from Q2

- risks: timeline slipping
- dependencies not resolved
- escalate if no response by Weds
```

Legible. Structure preserved. Uncertain words marked.

---

## Model benchmark — what actually works for handwriting

I tested several models via an OpenAI-compatible gateway. Results:

| Tier | Model | Notes |
|------|-------|-------|
| 🥇 Best | Claude Sonnet / Claude Opus (Anthropic API) | Accurate, preserves structure, handles upside-down |
| 🥉 Partial | GPT-4o (Azure OpenAI) | Works but less accurate on cursive |
| ❌ Broken | Gemini 2.5 Flash/Pro (via Vertex gateway) | Image bytes don't reach the model — returns text-only response |

Two key findings:
1. **Don't resize before sending**. I tried resizing to 1024px to reduce payload size. GPT-4o's accuracy dropped significantly. Send full resolution — Claude handles it fine.
2. **Gemini image routing is gateway-dependent**. Gemini works fine via Google's own API. Via an OpenAI-compatible proxy, the image payload may be silently dropped. Check your gateway's multimodal support before assuming it works.

---

## Two commands, you choose

I ended up with two slash commands in the OMP extension:

**`/scan`** — fast, no API cost, inline vision:
1. NAPS2 scans → `scan-<ts>.jpg`
2. `@path` attached to editor
3. You type your question, I read the image

**`/scan-ocr`** — transcription saved to disk:
1. NAPS2 scans → `scan-<ts>.jpg`
2. Python calls vision API → `scan-<ts>.txt`
3. Both `@image` and `@txt` attached

`/scan` is right for interactive Q&A — the vision model in the chat reads the image directly, no extra API call. `/scan-ocr` is right when you want a persistent text file for search or reuse.

Context window note: a 300 DPI scan (~2550×3300px) costs ~1,800 tokens as an image. The equivalent text transcript is ~300 tokens. If you're referencing the same scan across many turns, the transcript wins.

---

## One placement quirk

Scan the page **face-down** with the **top edge at the arrow mark** on the glass. If you place it the other way, the image comes out upside-down. Claude reads it correctly anyway (it notes the orientation and corrects mentally), but it's cleaner to get it right.

---

## Source

- [github.com/ankitg12/omp-scan](https://github.com/ankitg12/omp-scan) — the OMP extension (private for now; the pattern is in this post)
- [NAPS2 CLI docs](https://www.naps2.com/doc/command-line)
- [Anthropic vision API](https://docs.anthropic.com/en/docs/build-with-claude/vision)
- [Microsoft 24H2 eSCL bug](https://learn.microsoft.com/en-us/windows/release-health/status-windows-11-24h2#3446msgdesc)
