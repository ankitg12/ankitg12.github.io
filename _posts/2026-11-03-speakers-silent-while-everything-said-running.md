---
layout: post
title: "Every Layer Said RUNNING. The Speakers Were Silent."
date: 2026-11-03
categories: linux debugging audio
series: "Moving to Linux"
---

YouTube played in the browser with the progress bar moving, the volume at 100%, nothing muted, and no sound from the laptop speakers. My coding agent had just been given a voice ([previous post](https://ankitg12.github.io/ai/agents/macos/2026/11/01/i-gave-my-agent-a-voice.html)), and its first spoken greeting was also silent. It took about 35 minutes, one real but unrelated routing fault, one wrong theory, and one shutdown that changed nothing. The fix was to hold the power button for 30 seconds with the charger unplugged.

This post records what each layer reported, why none of them could see the fault, and the 4-second test that I now run first.

## The machine

Dell Pro 16 Plus (PB16255), BIOS 1.16.0, [Ubuntu](https://ubuntu.com/) with the `6.17.0-1033-oem` kernel, [PipeWire](https://pipewire.org/) with [WirePlumber](https://pipewire.pages.freedesktop.org/wireplumber/) 0.4.17. The internal audio is a Realtek RT722 codec on a [MIPI SoundWire](https://www.mipi.org/specifications/soundwire) bus. The RT722 is an SDCA device: the kernel log lists its functions at boot.

```text
acpi device:27: SDCA function UAJ (type 6) at 0x1
acpi device:28: SDCA function SmartMic (type 3) at 0x2
acpi device:29: SDCA function HID (type 10) at 0x3
acpi device:2a: SDCA function SmartAmp (type 1) at 0x4
```

`SmartMic` is the built-in microphone. `SmartAmp` is the speaker amplifier. Both are inside the same chip, so a working mic does not prove a working speaker.

## Timeline

| Time | Event |
|---|---|
| 2 Nov, day | Audio call on this laptop; sound worked |
| 2 Nov 17:45 | Suspend (s2idle) |
| 3 Nov 10:19 | Resume after 16.5 h |
| 3 Nov 10:25 | Power off; boot at 10:26 |
| 3 Nov 12:19 | Agent's spoken greeting: no sound |
| 3 Nov 12:23 | YouTube: no sound |
| 3 Nov 12:29 | Full shutdown, charger connected; boot 12:30. Still silent |
| 3 Nov 12:53 | Charger and dock unplugged, power button held 30 s |
| 3 Nov 12:54 | Boot. Sound works |

The boot list from `journalctl --list-boots` matches the table:

```text
 -4 ... Mon 2026-11-02 11:03:03 IST Tue 2026-11-03 10:25:51 IST
 -3 ... Tue 2026-11-03 10:26:46 IST Tue 2026-11-03 12:29:40 IST
 -2 ... Tue 2026-11-03 12:30:53 IST Tue 2026-11-03 12:52:39 IST
 -1 ... Tue 2026-11-03 12:53:19 IST Tue 2026-11-03 12:53:40 IST
  0 ... Tue 2026-11-03 12:54:35 IST
```

Boot `-1` lasted 21 seconds. The laptop appears to have started while the power button was held, and logind logged `Power key pressed short.` at 12:53:34 and shut it down again. That is expected with an internal battery; keep holding the button.

## Layer by layer: what each one said

### 1. The application stream

The first finding was real but not the cause. The browser stream was on the USB-C dock's audio output, not on the laptop speaker. On 30 September I had seen this same dock take the default output, because its USB Audio sink has a higher `priority.session` than the internal speaker. A WirePlumber Lua rule was installed then to keep the speaker as the default. On 3 November the dock was the default again, with the rule still in place.

I moved the stream to the speaker and later disabled the dock and HDMI audio profiles, so only the laptop outputs remain. That fixed the routing. It did not bring the sound back.

```text
$ pactl get-default-sink && pactl list short sink-inputs
alsa_output.pci-0000_c5_00.5-platform-amd_sdw.HiFi__Speaker__sink
525  58  524  PipeWire  float32le 2ch 44100Hz
726  58  725  PipeWire  float32le 2ch 48000Hz
```

Disabling profiles has a side effect. For a moment the saved default sink was `auto_null`, a dummy sink with no device behind it. Setting the default again with `pactl set-default-sink` fixed it, and `~/.local/state/wireplumber/default-nodes` then held the speaker. If you disable profiles, check that file.

### 2. Volume and mute

```text
$ pactl get-sink-volume @DEFAULT_SINK@ && pactl get-sink-mute @DEFAULT_SINK@
Volume: front-left: 65536 / 100% / 0.00 dB,   front-right: 65536 / 100% / 0.00 dB
        balance 0.00
Mute: no
```

The ALSA mixer agreed: `Speaker [on]`, `rt722 FU06` at 100% and `[on]`.

### 3. The PCM device

PipeWire state is one layer above the hardware, so I read the kernel's own PCM status during playback:

```text
state: RUNNING
hw_ptr: 55686032 → 55710160      (two reads 0.5 s apart)
```

At 48 kHz, 24,128 frames is 0.503 s. The DMA pointer moved at real time. The kernel was sending samples to the codec.

### 4. The SoundWire device

```text
/sys/bus/soundwire/devices/sdw:0:0:025d:0722:01/power/runtime_status   active
```

The codec was enumerated and awake.

### 5. The kernel log

I compared the audio lines of the silent boots (`-3`, `-2`) with the boot of 2 November (`-4`, when the call worked) and the fixed boot (`0`). They are the same: the four SDCA function lines and the headset jack input. No probe errors, no SoundWire timeouts, no ASoC warnings. The only difference was the card number of the headset jack (`card2` with the dock attached, `card1` without it).

The silent boot `-3` did have one kernel `WARNING` in `typec_altmode_update_active` (USB Type-C). The boot of 29 October has the same warning, so it is not unique to the silent boots.

**Every software layer reported a healthy playback path.** The fault was below what the driver can observe.

## The wrong turn

With all layers clean, I suspected the kernel. The kernel had changed from `-1032` to `-1033` on 13 October, and an upstream bug matches the symptom exactly: [Internal speakers silent on Dell 16 Plus (DB16250) despite fully working driver/mixer chain](https://github.com/thesofproject/linux/issues/5845). Realtek's fix, [ASoC: rt722: reset codec to fix abnormal sound](https://github.com/torvalds/linux/commit/d0ecbedd6a70f2fe768d46de740265dc8a4424e3), is not in this Ubuntu kernel's changelog.

The theory was wrong. I had a working audio call the day before, on the same `-1033` kernel. Ask "when did it last work, and what was the same then?" before you suspect the newest change.

The upstream commit message was still the best clue:

> The audio output may become abnormal after a warm reboot from Windows. Reset the codec once during hardware initialization to restore it to a known state and prevent the issue.

The RT722 can keep a bad internal state across an OS reboot, because it does not lose power. That changes the question from "which software layer is wrong" to "how do I remove the power from the codec".

## Stop listening, start measuring

"Can you hear it now?" is a slow and unreliable test. The person must be at the laptop, the room must be quiet, and frustration makes every answer less sure. The laptop has a microphone a few centimetres from its speakers, so I used it.

The test plays a 1 kHz tone on the speaker sink and records the built-in mic at the same time. It also records a silent baseline first. It measures the level at exactly 1 kHz with the [Goertzel algorithm](https://en.wikipedia.org/wiki/Goertzel_algorithm), a single-bin DFT that needs no NumPy:

```python
def goertzel(x, f, sr=48000):
    k = 2 * math.cos(2 * math.pi * f / sr)
    s1 = s2 = 0.0
    for v in x:
        s1, s2 = v + k * s1 - s2, s1
    return math.sqrt(max(0.0, s1*s1 + s2*s2 - k*s1*s2)) / len(x)
```

Silent speakers, after the 12:30 boot:

```text
no tone  : {'samples': 65536, 'rms': 116.3, 'at_1000Hz': 0.72}
1kHz tone: {'samples': 65536, 'rms': 110.7, 'at_1000Hz': 0.48}
```

The mic worked, because it recorded room noise (RMS about 110). The 1 kHz level did not go up while the tone played. With the PCM running and the DMA pointer moving, this measurement located the fault after the samples reached the codec.

## The fix: drain the residual power

A normal shutdown at 12:29 did not help. The charger was connected, so the board kept power, and the codec state survived.

Dell calls the fix a hard reset or [flea power drain](https://www.dell.com/support/kbdoc/en-us/000139016/how-to-perform-hard-reset):

1. Shut down.
2. Unplug the charger and the dock.
3. Hold the power button for 30 seconds. If the laptop starts, keep holding.
4. Connect the charger and start the laptop.

After the 12:54 boot, the same test:

```text
no tone  : {'samples': 143040, 'rms': 579.4, 'at_1000Hz': 1.45}
1kHz tone: {'samples': 144960, 'rms': 1120.9, 'at_1000Hz': 548.62}
```

The 1 kHz level went from 0.48 (silent) to 548.62 (working). The ratio to the baseline went from 0.7× to about 380×. There is no doubt left between those two numbers.

## Not fixed, prevented from wasting time: `speaker-check`

I cannot yet stop the codec from getting stuck. I can make the diagnosis take 4 seconds instead of 35 minutes. The test is now a script, `speaker-check`, which uses only the Python standard library and `pactl`, `paplay` and `parec` (they work on PipeWire through `pipewire-pulse`). It gives one of three verdicts:

| Verdict | Exit | Meaning | Action |
|---|---|---|---|
| `PASS` | 0 | The mic hears the speaker | Look at the app or its output device |
| `ROUTE` | 2 | Default output is not the laptop speaker | `pactl set-default-sink <speaker>` |
| `SILENT` | 3 | Route and data are fine, no sound comes out | Drain residual power, as above |

```text
$ speaker-check
sink           alsa_output.pci-0000_c5_00.5-platform-amd_sdw.HiFi__Speaker__sink
source         alsa_input.pci-0000_c5_00.5-platform-amd_sdw.HiFi__Mic__source
default_sink   alsa_output.pci-0000_c5_00.5-platform-amd_sdw.HiFi__Speaker__sink
samples        [74880, 92160]
baseline_1khz  0.22
tone_1khz      2569.62
ratio          11715.6
verdict        PASS
why            the mic hears the speaker
```

The pass threshold is a ratio of 10. The measured silent ratio was 0.7 and the measured working ratios were about 380 and 11,700, so the threshold has a wide margin on both sides.

## The other dialog: "The login keyring did not get unlocked"

After the 12:54 boot, GNOME asked for the login keyring password. This is not related to the audio. The PAM log shows the cause:

```text
12:54:49 gdm-fingerprint: gkr-pam: no password is available for user
12:54:52 gdm-password:    gkr-pam: couldn't unlock the login keyring.
```

The keyring is encrypted with the login password. A fingerprint login does not give PAM a password, so `gkr-pam` cannot unlock the keyring. On the boots where I typed the password, the same log says `gkr-pam: unlocked login keyring`. Type the password at the first login after a boot and use the fingerprint for the screen lock. ([How the keyring holds my secrets]({% post_url 2026-10-08-secrets-in-the-keyring-moving-to-linux %}).)

## What I take from it

1. **Software health is not acoustic output.** Route, volume, mute, PCM state, DMA pointer and device power can all be correct while the amplifier is silent. Only a measurement in the air proves sound.
2. **Measure with what is already there.** The built-in mic is a free acoustic sensor. Twenty lines of Python turned "can you hear it now?" into a number.
3. **"Restart" does not mean "power cycle".** Here a shutdown with the charger connected did not clear the fault, and a shutdown with all power removed did. My reading is that the codec kept its state on a rail that stayed powered; I did not measure the rail.
4. **Anchor on the last known good state.** The newest change (the kernel) was not the cause, because audio worked on that kernel the day before.

## Not checked

- What put the codec in the bad state. The 16.5-hour s2idle suspend before the silent boots is a candidate. I have no evidence that it is the cause.
- Whether the speakers were already silent between the resume at 10:19 and the power off at 10:25.
- Whether a kernel with Realtek's reset commit would clear the state without the power drain.
- Why the WirePlumber priority rule did not keep the speaker as the default. My guess is that a default saved in `default-nodes` takes precedence over `priority.session`; I have not tested it.
- Whether the audio call on 2 November used the laptop speakers or a headset.

## Source

- [thesofproject/linux#5845 — Internal speakers silent on Dell 16 Plus despite fully working driver/mixer chain](https://github.com/thesofproject/linux/issues/5845)
- [ASoC: rt722: reset codec to fix abnormal sound (d0ecbedd)](https://github.com/torvalds/linux/commit/d0ecbedd6a70f2fe768d46de740265dc8a4424e3)
- [Dell: How to perform a hard reset](https://www.dell.com/support/kbdoc/en-us/000139016/how-to-perform-hard-reset)
- [Goertzel algorithm](https://en.wikipedia.org/wiki/Goertzel_algorithm)
- [PipeWire](https://pipewire.org/), [WirePlumber](https://pipewire.pages.freedesktop.org/wireplumber/), [MIPI SoundWire](https://www.mipi.org/specifications/soundwire)
- Earlier audio post in this series: [The Mic Exists. The Firmware Doesn't Declare It.]({% post_url 2026-10-20-mic-exists-firmware-does-not-declare-it %})
