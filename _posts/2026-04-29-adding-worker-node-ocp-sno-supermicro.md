---
layout: post
title: "Adding a bare-metal worker node to OpenShift SNO on Supermicro: 6 gotchas that cost me 4 hours"
date: 2026-04-29
categories: openshift kubernetes devops baremetal
---

Today I spent 4 hours adding a single worker node to an OpenShift Single Node (SNO) cluster running on Supermicro H13 hardware with 8x AMD MI350X GPUs. The task should take 20 minutes. Here's what went wrong and how to avoid it.

**TL;DR**: Use the [Red Hat Assisted Installer portal](https://console.redhat.com/openshift) to generate a discovery ISO. Boot it via BMC virtual media. Approve the host in the portal. Done. Everything else below is what happens when you don't.

---

## Gotcha 1: `Cd` vs `UsbCd` — 60 minutes lost

On Supermicro servers, BMC virtual media appears as **UEFI USB CD/DVD (ATEN Virtual CDROM)** — not as a regular CD/DVD drive.

```
Boot order:
  1. UEFI Hard Disk: ubuntu
  2. UEFI CD/DVD               ← physical drive (empty)
  3. UEFI Network
  4. UEFI USB CD/DVD: ATEN Virtual CDROM  ← this is your virtual media
```

When you set `BootSourceOverrideTarget: Cd` via Redfish, the BIOS boots from entry 2 — the empty physical drive — and falls through to Ubuntu. The BMC still streams the ISO (BIOS enumerates the virtual device during POST) so your nginx logs show `206 Partial Content` and you think it's working. It's not.

**Fix**: Use `BootSourceOverrideTarget: UsbCd`.

## Gotcha 2: Legacy boot mode — RHCOS is UEFI-only

Supermicro defaults `BootSourceOverrideMode` to `Legacy` when you don't specify it. RHCOS requires UEFI. The ISO boots silently in Legacy mode, fails to find a bootloader, and the system falls back to the next boot device.

**Fix**: Always set `BootSourceOverrideMode: UEFI` explicitly in your Redfish PATCH.

```python
# Correct Redfish PATCH body
{"Boot": {
    "BootSourceOverrideEnabled": "Once",
    "BootSourceOverrideTarget": "UsbCd",   # not Cd
    "BootSourceOverrideMode": "UEFI",       # not Legacy
}}
```

## Gotcha 3: Python SimpleHTTP won't work for BMC virtual media

Supermicro BMC fetches ISOs using HTTP range requests (`Range: bytes=X-Y`). Python's built-in `http.server` (SimpleHTTPServer) doesn't support range requests and returns 200 with the full file or 404 — either way the BMC fails to stream it.

**Fix**: Use nginx.

```bash
podman run --rm -d --name iso-serve -p 9002:80 \
  -v ~/iso-serve:/usr/share/nginx/html:ro,z \
  docker.io/nginx:alpine
```

Port 9002 avoids conflicts with OpenShift's internal services.

## Gotcha 4: MCS ignition is v2.2.0, RHCOS 9.x needs v3.0+

If you try to manually fetch the worker ignition from the Machine Config Server (port 22623):

```bash
curl -sk https://<control-plane>:22623/config/worker > worker.ign
```

You get ignition v2.2.0. RHCOS 9.x requires ignition v3.0+. The node boots into emergency mode with:

```
Ignition has failed. Note that only Ignition spec v3.0.0+ configs are accepted.
```

**Fix**: Use `oc extract secret/worker-user-data -n openshift-machine-api --keys=userData --to=-` which gives a v3.x pointer ignition. Or better, use the Assisted Installer.

## Gotcha 5: The api-int DNS chicken-and-egg problem

The pointer ignition from `oc extract` references:
```
https://api-int.<cluster>.<domain>:22623/config/worker
```

On SNO clusters, `api-int` only exists in the cluster's internal CoreDNS — not resolvable from bare-metal nodes during bootstrap. So you think: add an `/etc/hosts` entry to the ignition!

This doesn't work. Ignition fetches the merge config **before** writing storage files. The hosts entry never exists when it's needed.

**Fix**: Use the Assisted Installer which handles all of this. If you must do it manually, fetch the full config from MCS (via localhost on the control plane), convert v2→v3 (remove the `filesystem` field from file entries, bump version), and embed it directly — no merge fetch needed.

## Gotcha 6: coreos-installer dies when your SSH session ends

If you run:
```bash
ssh d07-31 'ssh core@<new-node> "sudo coreos-installer install /dev/nvme0n1 ..."'
```

When your outer session ends, the pipe dies, killing the inner SSH, killing coreos-installer mid-write. You've now partially written RHCOS to a disk that may not boot.

**Fix**: Run coreos-installer in `tmux` **on the target node** directly, not via double-SSH chains.

```bash
# SSH directly to the live RHCOS node
ssh core@<new-node>

# Run inside tmux on the target
tmux new-session -d -s install \
  "sudo coreos-installer install /dev/nvme0n1 --ignition-file /tmp/w.ign --insecure && sudo reboot"
```

---

## The right way: Assisted Installer

All of the above complexity disappears:

1. Go to [console.redhat.com/openshift](https://console.redhat.com/openshift) → your cluster → **Add hosts**
2. Download the discovery ISO (~130MB minimal)
3. Serve via nginx, mount via BMC, boot with `UsbCd` + UEFI
4. Host appears in the portal → approve → portal handles install + CSR approval

Or via CLI:
```bash
# Install ocm-cli
curl -sL https://github.com/openshift-online/ocm-cli/releases/download/v1.0.14/ocm-linux-amd64 -o /usr/local/bin/ocm
chmod +x /usr/local/bin/ocm

# Login and manage
ocm login --token=<offline-token>  # from console.redhat.com/openshift/token
ocm get /api/assisted-install/v2/hosts
```

---

## Tools built during the session

- **[bmc.py](https://github.com/pensando/gpu-operator-qa/blob/main/scripts/bmc/bmc.py)**: Supermicro Redfish CLI — virtual media, boot override, power, screenshot capture, SOL snapshot, event log. The `screenshot` subcommand saved a lot of time: `python3 bmc.py <host> screenshot` captures the console without opening the iKVM browser.

```
bmc.py <host> vm-status
bmc.py <host> vm-mount <http-url>
bmc.py <host> boot-once UsbCd        # always UsbCd, always UEFI
bmc.py <host> boot-info              # verify before reset
bmc.py <host> power ForceRestart
bmc.py <host> screenshot             # capture console to PNG
bmc.py <host> log -n 10             # BMC event log
```

> Generated with [Claude](https://claude.ai) (Anthropic) based on a real session with @ankitg12.
