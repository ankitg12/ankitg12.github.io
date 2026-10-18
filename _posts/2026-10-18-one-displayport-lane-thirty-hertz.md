---
layout: post
title: "WIP: the monitor is online at 30 Hz because one lane trained"
date: 2026-10-18 14:50:00 +0530
categories: macos displays
---

**Status: work in progress.** The panel is online. The link is not fixed. This note stops at the evidence. It will be updated when a different cable or the other port trains both lanes, or when that test fails.

**Problem**

The external monitor dropped, came back, and dropped again. The keyboard on the same USB-C hub did not follow it. A replug that made the picture return left the panel at 30 Hz. That is not a healthy attach.

The machine is a MacBook Air with an M1. The panel is an MSI MP223 E2, rotated 270°. The hub is a USB-C device, not a Thunderbolt dock. Both Thunderbolt receptacles reported no device while the panel was up, and again while it was dark.

**Two paths on one plug**

The keyboard and the picture do not share a failure. The keyboard is a USB 2 device at 12 Mbps (Sino Wealth, vendor id 9610, product id 315 — the same identifiers as in {% post_url 2026-10-18-tilde-key-prints-plus-minus %}). DisplayPort alt mode uses the SuperSpeed pairs. USB 2 can stay enumerated while those pairs train zero lanes.

| Moment | What macOS listed | DisplayPort | Plug events |
|---|---|---|---|
| Panel dark, keyboard up | Built-in LCD only | Hotplug high, sink count 1, rate "No Link", 0 lanes | 15 |
| After a USB-C replug the Mac counted | MSI online, `1080 x 1920 @ 30.00Hz` | Hotplug high, 2.7 Gbps (HBR), 1 lane | 18 |

`MaxLaneCount` was 2 in both rows. USB 3 was active on the port, so alt mode never offered four DisplayPort lanes. The bug is not "the Mac refused a 4-lane dock". The bug is that one of the two lanes that were offered did not lock, and for a while none of them did.

An earlier replug did not move `Plug Event Count`. It stayed at 15, hotplug stayed high, and the lane count stayed 0. The connector had not electrically left the port. Reseating is not a reset unless that counter moves.

**Why 30 Hz**

After the counted replug:

```text
LinkRateDescription = 2.7 Gbps (HBR)
LaneCount = 1
MaxLaneCount = 2
UI Looks like: 1080 x 1920 @ 30.00Hz
```

HBR is 2.7 Gbit/s per lane before 8b/10b. One lane therefore carries about 2.16 Gbit/s of payload. A 1920×1080 60 Hz 8-bit picture needs about 3.56 Gbit/s. The 30 Hz timing needs about half of that, and it fits. Two HBR lanes would have been enough for 60 Hz. One lane is why the "working" mode is 30 Hz, and why the same train falls back to no link.

The partner identity on that port reported vendor id 0 and product id 4114. SuperSpeed came up as Gen 1 while the picture was absent, and as Gen 2 after the counted replug. Low Power Mode was on, on AC power. It was not the mechanism: the port was not suspended, hotplug was high, and the keyboard was enumerated.

The HDMI cable between the hub and the panel can pass this 30 Hz picture. The black screen cleared on a USB-C reseat, not on an HDMI reseat. That does not prove the HDMI cable is good. It does put the failed train on the USB-C alt-mode link.

**Check**

```bash
system_profiler SPDisplaysDataType SPThunderboltDataType
ioreg -p IOUSB -w0
ioreg -r -n 'Port-USB-C@2' -w0 -l
ioreg -r -c IOPortTransportStateDisplayPort -w0 -l
```

Read these fields, not the glowing panel:

```text
Plug Event Count
ConnectionCount
HPD_StateDescription
LinkRateDescription
LaneCount
MaxLaneCount
```

Online at 30 Hz with `LaneCount` 1 is the broken state. The open test is a different USB-C cable, then the other port on the Mac. Record the same fields. Both lanes at a higher rate, and a mode above 30 Hz, is the fix. Another 30 Hz attach is the same bug.

## Source

- [Connect a display to your Mac](https://support.apple.com/guide/mac-help/connect-a-display-to-your-mac-mchl7c7ebe9a/mac)
- [DisplayPort over USB-C](https://www.displayport.org/displayport-over-usb-c/)
- Same keyboard, different fault: {% post_url 2026-10-18-tilde-key-prints-plus-minus %}
