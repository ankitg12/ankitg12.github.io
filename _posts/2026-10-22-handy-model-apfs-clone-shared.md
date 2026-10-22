---
layout: post
title: "One 698 MB Transcription Model, Two Mac Accounts, Zero Extra Disk"
date: 2026-10-22
categories: macos ai productivity
---

Two user accounts on one Mac. Both want local voice dictation. The transcription model is 698 MB. Downloading it twice is wasteful; copying it twice is worse on a 85%-full disk. The fix is one APFS clone and a five-line script.

The app is [Handy](https://github.com/cjpais/Handy) — free, open-source, runs Whisper/Parakeet models locally. I installed it on my account; the second account needed the same model without a second download.

## Where the model actually lives

Not where you'd guess. Handy's support folder has an empty `models/` directory:

```
~/Library/Application Support/com.pais.handy/
├── models/            ← empty
├── settings_store.json
├── history.db
└── recordings/
```

The real weights sit in the Hugging Face hub cache:

```
~/.cache/huggingface/hub/models--handy-computer--parakeet-unified-en-0.6b-gguf/
├── blobs/
│   └── 4b50b6dd862bf6e34...   ← 698 MB, filename IS the sha256
├── refs/
└── snapshots/<commit>/parakeet-unified-en-0.6b-Q8_0.gguf  ← symlink into blobs
```

The log line that confirms what's loaded: `model 'handy-computer/parakeet-unified-en-0.6b-gguf/...', backend 'MTL0'` — Metal, on-device. The blob filename is its own content hash ([model on HF](https://huggingface.co/handy-computer/parakeet-unified-en-0.6b-gguf)), so filename match across accounts is byte-equality, no hashing ceremony needed.

My first instinct was a tar in `/Users/Shared`. Wrong — tar duplicates 698 MB for nothing. A plain `cp` is no better: same bytes twice on disk. The right op on APFS is a clone.

## The share

```sh
mkdir -p /Users/Shared/handy/hub
cp -cR ~/.cache/huggingface/hub/models--handy-computer--parakeet-unified-en-0.6b-gguf \
  /Users/Shared/handy/hub/
```

`-c` is the whole trick: copy-on-write clone, zero new blocks. `du` reports 698 MB in both places (logical size); `df` doesn't move (physical size). One physical copy, two logical references.

The seed script the second account runs (`/Users/Shared/handy/setup-handy-model.sh`):

```sh
#!/bin/sh
set -eu
SHARED_MODEL_DIR="/Users/Shared/handy/hub/models--handy-computer--parakeet-unified-en-0.6b-gguf"
DEST="$HOME/.cache/huggingface/hub"
mkdir -p "$DEST"
cp -cR "$SHARED_MODEL_DIR" "$DEST/"
```

Open Handy once after; it finds the cached model and never hits the network.

## Why a clone per account, not a symlink to Shared

A symlink saves even the logical 698 MB. I didn't do it: Handy writes lock files next to the blobs, so the target account needs write access into `/Users/Shared`, and deleting the shared copy later silently breaks the account. A clone costs zero physical bytes and keeps each account independent. The only shared-state risk — a future model revision diverging per account — is exactly what you want: each account re-clones the new revision once.

## What not to share

`settings_store.json` stays per-account. It holds hotkeys and onboarding state tied to whoever configured first; copying it clobbers the second account's key setup. Share the weights, not the preferences.

## Verification

From the first account, with `sudo -u`:

```sh
$ sudo -u seconduser ls -l ~seconduser/.cache/huggingface/hub/models--handy-computer--*/blobs/
-rw-r--r--@ 1 seconduser staff 731357568 4b50b6dd862bf6e346929aaf4f5eaacec003bfa3f56462d6c874b41ef2f38795
```

Same name, same 731,357,568 bytes — same bits, guaranteed by HF's content-hash naming. And `df` before/after:

```
# tar present:              229G  195G   34G  86% /
# tar deleted + clone live: 229G  194G   35G  85% /
```

The tar I built first cost a full copy; the clone costs nothing measurable.

## The pattern

Any multi-account Mac sharing large immutable-ish blobs — models, SDKs, asset packs — gets the same treatment: put one clone in `/Users/Shared`, seed per-account with `cp -cR`, never copy or archive. Downloads happen once, disk pays once, accounts stay independent.
