---
layout: post
title: "Diagnosing a delayed Wi-Fi failure: correlating power events with NCSI in netsh wlan show wlanreport"
date: 2026-07-09
categories: windows networking tools
---

`ping` timed out three times, then failed outright with `General failure`, then worked again two minutes later after a Wi-Fi radio toggle. The obvious read is "Wi-Fi hiccup, restart fixed it." The actual mechanism took three iterations of Windows event-log correlation to pin down, and the tool that finally cracked it — `netsh wlan show wlanreport` — is one most people have never run.

## The known bug, and the part that isn't documented

The base symptom is not new. [A Microsoft Q&A thread from 2024](https://learn.microsoft.com/en-us/answers/questions/3849534/windows-11-doesnt-renew-ips-after-resuming-from-mo) — 136 responses, still open — reports Windows 11 failing to renew DHCP leases after resuming from Modern Standby, with `ipconfig /all` showing only a self-assigned address until a manual `ipconfig /renew`. Workarounds floated in that thread (disabling Fast Startup, `netsh int ip reset`) are unverified folk remedies; no confirmed fix exists.

What that thread describes is the *severe* version: no connectivity at all until manual intervention. What I hit was quieter and, as far as I can find, undocumented: **the connection looked completely healthy immediately after resume, then failed silently 11–12 minutes later**, self-healed within seconds, and would have gone unnoticed entirely if `ping` hadn't happened to need a fresh ARP resolution at the unlucky moment.

Proving that causal chain — resume now, failure minutes later — needed more than a disconnect log. It needed power events and connectivity events in the same timeline.

## Three iterations, three wrong-ish answers

**First pass**, from `Microsoft-Windows-NCSI/Operational` alone: two `SuspectArpProbeFailed` events, self-healed in seconds. Conclusion: "a Wi-Fi toggle forced fresh ARP resolution, fixing a stale entry." Plausible, and wrong about the mechanism — nothing in that log says *why* the ARP probe failed.

**Second pass**, widening to 24 hours across `WLAN-AutoConfig`/`NetworkProfile`/`NCSI`/`DNS-Client`: found 7 similar capability-degrade events on the same interface over ~14 hours, not an isolated incident. Better, but still no explanation for *why* — just that it recurs.

**Third pass**, `netsh wlan show wlanreport`, changed the picture entirely.

## What `wlanreport` has that a custom log script doesn't

```
netsh wlan show wlanreport   # requires an elevated prompt
```

This generates an HTML report at `C:\ProgramData\Microsoft\Windows\WlanReport\wlan-report-latest.html` covering the last 3 days: disconnect reasons in plain English, session durations, adapter details — genuinely useful, and [already covered well by a few 2026 write-ups](https://www.makeuseof.com/windows-wifi-report-shows-disconnect-history/) if that's all you need.

The part those write-ups don't mention: it also pulls **NDIS/Kernel power events** (event ID 10311 — `D0_SystemResume`, `Dx_SystemSleep`, `D0_NicActive`) into the same timeline as the WLAN and NCSI events. A hand-rolled `pywin32` script pulling `Microsoft-Windows-WLAN-AutoConfig/Operational` and `Microsoft-Windows-NCSI/Operational` — which is what I'd built for the first two passes — simply doesn't touch the power-event channel unless you know to ask for it specifically. `wlanreport` bundles it by default.

Laying the two event families side by side in the same report:

```
06:32:57  D0_SystemResume       <- laptop wakes from Modern Standby
06:32:57  Wireless security re-keys, CapabilityReset
06:32:58  Capability: Internet restored (looks completely healthy)
06:44:04  SuspectArpProbeFailed  <- 11 minutes, 7 seconds later
06:44:27  SuspectArpProbeFailed (retry)
06:44:45  Capability: Internet restored (self-healed)
06:45:08  SuspectDnsProbeFailed
06:45:15  Capability: Internet restored (self-healed)
06:48:37  Manual Wi-Fi disconnect/reconnect (the recovery action)
```

Every capability-degrade event that morning falls within roughly 11–12 minutes of a `D0_SystemResume`. That's not a coincidence sample size of one — the same pattern repeated at three other points across a wider 24-hour pull, each a similar interval after a resume event.

## The mechanism, and why it's delayed rather than immediate

Per [Microsoft's Modern Standby design docs](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/modern-standby-network-connectivity), Adaptive Connected Standby deliberately quiesces network activity during sleep and restores it "instantly" on resume — that's the intended behavior, and NCSI's own logs confirm the radio re-associates and reports `Internet` capability within about a second of waking. The problem isn't the resume itself; it's that ARP cache and/or DHCP lease state doesn't get a clean, immediate refresh alongside that fast radio re-association. The mismatch doesn't surface as a failure until something actually needs a fresh ARP or DNS resolution — which can be minutes later, whenever the next such request happens to fire. That's the delay: not the network taking 11 minutes to break, but 11 minutes elapsing before anything asked a question the stale state couldn't answer.

This also explains why a long-lived connection (an open Slack session, for instance) can keep working straight through the same window: it's reusing an already-cached gateway MAC and tolerant TCP retransmission timers, not issuing a fresh ARP request that would expose the gap.

## Why this needed the correlation, not either log alone

NCSI alone tells you *that* something failed and briefly why (`ChangeReason: SuspectArpProbeFailed`). WLAN-AutoConfig alone tells you about actual disassociation events — and in this case, there wasn't one during the failure window, which is itself informative (rules out a real radio drop). Neither log, on its own, points at Modern Standby as the trigger. Only holding the power-event timeline next to the connectivity-event timeline — and noticing the fixed ~11-minute offset repeat across multiple incidents — makes the causal link visible. That correlation step is the actual contribution here; the underlying DHCP-after-resume bug was already known.

## Update, 29 July 2026: recurrence exposed a lower-level power problem

Twenty days later, the laptop suffered more frequent Wi-Fi failures. The observable failure mode was different: on 9 July the radio remained associated while the packet path temporarily lost the gateway; on 29 July, WLAN events explicitly reported that the **driver disconnected** across multiple SSIDs.

The newer evidence also included daily NDIS event 10317 failures for `Microsoft Wi-Fi Direct Virtual Adapter #2`: Windows tried to return the adapter to operational power, and the transition failed. That made a driver/power-state interaction considerably more likely than the ARP-only evidence could establish on 9 July.

The distinction matters because the original Wi-Fi off/on action was a **recovery**, not a persistent repair. It reset the interface and restored traffic, but no driver, wireless-power, or PCIe power-management setting was changed. Before the 29 July repair, the live configuration still had:

- Wireless adapter power saving on battery: **Medium Power Saving**
- PCI Express Link State Power Management on AC: **Maximum Power Savings**
- PCI Express Link State Power Management on battery: **Maximum Power Savings**

The persistent mitigation applied on 29 July was:

- Wireless adapter power saving on battery: **Maximum Performance**
- PCI Express Link State Power Management on AC and battery: **Off**

After activating the updated power plan, the link was healthy at 866.7 Mbps with 89% signal, and a 30-packet Internet test completed with zero loss. The installed Realtek RTL8852BE driver, `6001.15.158.600` dated 25 April 2025, was already newer than HP's publicly listed `6001.15.154.0` package, so blindly replacing it with the public package would have been a downgrade rather than an update.

The packet-path failure then recurred at 10:51—after those policies were active. GlobalProtect's network profile disconnected at 10:51:40; eight seconds later, NCSI logged `SuspectArpProbeFailed` on the physical Wi-Fi interface, followed by another failed probe at 10:52:11. WLAN security restarted without an 8003 disassociation event, Internet capability returned at 10:52:17, and GlobalProtect reconnected at 10:58:29. There had been no system-resume event in the preceding two hours and no NDIS 10317 power-transition failure in the preceding hour.

That timeline confirms a recurrence of the 9 July **associated-but-broken packet path**, not the separate full driver-disconnect mode seen earlier on 29 July. It also shows that disabling PCIe link-state power management did not prevent this non-resume instance, although the policy may still mitigate the distinct driver-disconnect failure.

An elevated HP Image Assistant analysis then completed successfully and offered no Realtek Wi-Fi driver update—only unrelated device and application packages. HP therefore had no model-specific Wi-Fi update to apply, and installing its older public package would still have been a downgrade. Hardware replacement is also premature: the adapter starts normally, live RF health is strong, and Windows has logged no WHEA hardware errors. The next recurrence needs a targeted Pktmon/ETW capture of ARP and NDIS traversal so the failing layer can be identified before changing more variables.

## Takeaway

If you're chasing an intermittent Wi-Fi issue on Windows and reaching for Event Viewer, start with `netsh wlan show wlanreport` before hand-rolling a query against individual event channels — it already bundles the power-event layer that's often the real trigger, and skipping straight to WLAN/NCSI logs alone (as I did on the first two passes) can lead to a plausible but incomplete diagnosis. If the failures in your own report cluster at a consistent offset after `D0_SystemResume`, you're very likely looking at the same Modern-Standby DHCP/ARP-refresh gap described in the Microsoft Q&A thread above — worth checking `powercfg /spr`'s "Networking in Standby" field as a next step to see whether your session even had networking enabled during sleep at all.

## Sources

- [Microsoft Q&A: Windows 11 doesn't renew IPs after resuming from Modern Standby](https://learn.microsoft.com/en-us/answers/questions/3849534/windows-11-doesnt-renew-ips-after-resuming-from-mo) — the known base symptom, unresolved as of writing
- [Microsoft Learn: Modern Standby network connectivity](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/modern-standby-network-connectivity) — Adaptive Connected Standby's intended behavior
- [MakeUseOf: Windows has a built-in Wi-Fi report that shows every disconnect](https://www.makeuseof.com/windows-wifi-report-shows-disconnect-history/) — good coverage of `wlanreport`'s disconnect-history features (not power-event correlation)
