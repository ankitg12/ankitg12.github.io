---
layout: post
title: "The Mic Exists. The Firmware Doesn't Declare It. Stop Writing modprobe Confs."
date: 2026-10-20
categories: linux debugging audio
series: "Moving to Linux"
---

I wanted voice typing on a spare Lenovo V15-IGL. PipeWire showed one source: `auto_null.monitor`. No mic. The fix the forums prescribe — `options snd-hda-intel dmic_detect=0` — would have been theater. The driver was already bound. There was no codec behind it.

## The use case

A working dictation stack lives on my other Linux box: mic → local Whisper → post-processing → typed into the focused window. I wanted the same on this spare laptop — talk, get text, skip the keyboard for first drafts. That needs exactly one thing from the OS: a capture device. `pactl list sources` returned only the null-sink monitor. So the session became: where is the mic?

## The machine

Lenovo V15-IGL (type 82C3), Celeron N4020, 3.6 GiB RAM, Ubuntu 22.10. Lenovo's PSREF spec sheet for this model lists, under Multi-Media: "Microphone: Single microphone", plus a 3.5mm headset/mic combo jack. Budget single analog mic, wired through the board audio codec. It should be there.

What Linux saw instead: only an `Intel Geminilake HDMI` codec at address 2. No analog codec anywhere.

```text
$ arecord -l
**** List of CAPTURE Hardware Devices ****
$ pactl list sources short
35	auto_null.monitor	PipeWire	float32le 2ch 48000Hz	SUSPENDED
```

## The obvious suspect (and why it was innocent)

Gemini Lake audio on Linux is a known swamp. The kernel often binds the wrong DSP driver (`snd_soc_skl`, `snd_soc_avs`) or hangs probing digital mics, leaving Dummy Output and no capture. The standard fix: force legacy HDA, blacklist the SOF/SST drivers, reboot.

I checked the binding first:

```text
$ lspci -nnk -s 00:0e.0
00:0e.0 Audio device [0403]: Intel Corporation Celeron/Pentium Silver Processor
  High Definition Audio [8086:3198] (rev 06)
	Kernel driver in use: snd_hda_intel
	Kernel modules: snd_hda_intel, snd_soc_skl, snd_soc_avs, snd_sof_pci_intel_apl
```

Legacy HDA was already bound. The wrong-driver theory was dead before I touched anything.

## What the kernel log actually said

```text
snd_hda_intel 0000:00:0e.0: bound 0000:00:02.0 (ops i915_audio_component_bind_ops)
snd_hda_intel 0000:00:0e.0: azx_get_response timeout, switching to polling mode
snd_hda_intel 0000:00:0e.0: No response from codec, disabling MSI
snd_hda_codec_hdmi hdaudioC0D2: Unable to sync register 0x2f0d00. -5
```

The codec bus timed out during enumeration. Only HDMI stubs got registered (`card 0: PCH [HDA Intel PCH], device 3/7/8/9/10: HDMI`). No analog paths, no capture. Identical on every boot since July 2023 — this never worked under this install, so it wasn't a regression.

## The live proof: rebind into silence

If the codec were present but misconfigured, a driver rebind would re-enumerate it. I tried:

```bash
echo 0000:00:0e.0 | sudo tee /sys/bus/pci/drivers/snd_hda_intel/unbind
echo 0000:00:0e.0 | sudo tee /sys/bus/pci/drivers/snd_hda_intel/bind
```

Result: `no codecs found!` — worse than before. Even the HDMI stubs went. A full PCI remove + rescan gave the same. The bus is electrically silent to the OS; only a cold boot re-probes it, and cold boots have been failing identically for years. Retried once with power-save disabled (`power_save=0`) in case the codec was napping. Same silence.

## What the firmware declares

I dumped the ACPI tables and searched them for codec declarations:

| Table | Audio declarations |
|---|---|
| DSDT `HDAS` node | HDMI endpoints only |
| DSDT `RTKC` reference | DisplayPort power-management node, not an audio codec |
| SSDT3 `INT34C1` / `INT343A` | "Intel SST Audio Codec" on I2C0 — both fail `_STA`, report absent |
| NHLT table | Missing entirely |
| I2C bus (`ESSX`/`8336`/`ALC`/`10EC`) | No codec device anywhere |

The SST codecs declare themselves and then report absent. No I2S codec, no HDA analog codec, no NHLT describing a mic topology. There is nothing for a modprobe option to claim — `dmic_detect=0` forces a driver that is already bound onto a bus with nothing answering.

## The fix I didn't apply

Writing the forum-prescribed conf would have changed nothing except risk: a bad quirk can only make the half-working HDMI audio worse, and every reboot to test it costs a full cycle on a 2-core Celeron. The evidence chain — bound driver, bus timeouts, silent rebind, absent `_STA`, no codec in any ACPI table — all points past software. Whether it's a stripped DSDT, a vendor-disabled SKU path, or a dead codec chip is indistinguishable from the OS side. All three end the same way: no software fix available.

## What unblocks voice input

A ₹300 USB sound dongle, or any USB/Bluetooth headset. USB audio doesn't touch the HDA bus at all — it enumerates as its own card with its own capture device. That one purchase restores the entire dictation stack: mic → Whisper → typed text. The internal mic stays a firmware dead end, and knowing that cost one session instead of a week of modprobe archaeology.

## The rule

Check the binding before the forums. `lspci -k` takes a second; if the right driver is already bound and the bus is silent, no module option will conjure hardware. The rebind test (`unbind`/`bind` + `aplay -l`) is the cheapest experiment available — if re-enumeration finds nothing, stop editing config and go around the bus instead.

## Source

- [Lenovo V15-IGL PSREF (PDF)](https://psref.lenovo.com/syspool/Sys/PDF/Lenovo/Lenovo_V15_IGL/Lenovo_V15_IGL_Spec.pdf) — "Microphone: Single microphone"
- [SOF project: NO SOUND for GeminiLake with ES8336 #3336](https://github.com/thesofproject/linux/issues/3336) — the quirk-trial rabbit hole this could have become
