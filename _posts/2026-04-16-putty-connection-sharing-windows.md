---
layout: post
title: "Fast SSH on Windows via PuTTY connection sharing"
date: 2026-04-16
categories: windows ssh infrastructure
---

Some lab nodes run a custom SSH authentication check on every new connection. It adds 7–8 seconds per `ssh` invocation — before you type anything. Annoying in isolation. Multiplied across a workday, it becomes a real tax.

On Linux the fix is two lines in `~/.ssh/config`:

```
ControlMaster auto
ControlPath ~/.ssh/cm/%r@%h
```

The first connection pays the cost; every subsequent one is instant — same TCP connection, new channel. [OpenSSH multiplexing](https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Multiplexing) is well-documented and widely used.

On Windows, OpenSSH's ControlMaster is silently broken. The config parses without error. No socket is ever created. Nothing multiplexes. This is a [known open issue](https://github.com/PowerShell/Win32-OpenSSH/issues/1328) in Win32-OpenSSH, also called out in the [VS Code Remote SSH tracker](https://github.com/microsoft/vscode-remote-release/issues/96).

---

## What else was tried

**PuTTY connection sharing with `-nc localhost:22` as ProxyCommand.** The idea: plink holds the upstream connection, and downstream `ssh` invocations proxy through it via a `direct-tcpip` channel to localhost:22. Timing: 13 seconds. The channel opens a new TCP socket on port 22 — the server sees a new connection, the auth check fires again.

**PuTTY sharing with `-N` upstream.** Run `plink -share -N ...` in the background, use `plink -share ... hostname` as the downstream. The downstream hangs indefinitely.

Reading [ssh/sharing.c](https://git.tartarus.org/?p=simon/putty.git;a=blob;f=ssh/sharing.c) revealed why. With `-N`, PuTTY opens no session channel. When a downstream sends `SSH2_MSG_CHANNEL_OPEN`, the upstream forwards it to the server — but the server never responds. No `CHANNEL_OPEN_CONFIRMATION` comes back. The downstream waits forever.

The PuTTY wishlist has a related item: [dedicated-sharing-upstream](https://www.chiark.greenend.org.uk/~sgtatham/putty/wishlist/dedicated-sharing-upstream.html) — a proposal for a `pshare` utility that would act as a persistent headless upstream. It has been open since 2013.

The fix is one word.

---

## What works

Start the upstream with a real session command:

```
plink -share -batch -load <session> "sleep infinity"
```

This opens an actual session channel. The server responds. Downstream connections can open additional channels on the same authenticated connection — no new TCP handshake, no auth check. Each downstream invocation takes about 1 second.

The auth check gates TCP connections, not SSH channels.

---

## The O'Reilly trick, adapted

*Linux Server Hacks* (Rob Flickenger, O'Reilly) describes a neat pattern: one script called `ssh-to`, symlinked under multiple names. It uses `basename $0` to know which host to connect to.

```sh
#!/bin/sh
ssh `basename $0` $*
```

```
$ ln -s ssh-to server1
$ ln -s ssh-to server2
```

On Windows, `%~n0` in a `.cmd` file gives the script's own name even through a symlink. Same idea, adapted:

**`plink-to.cmd`** — the master shim:
```batch
@echo off
python "%~dp0plink-to.py" "%~n0" %*
```

**`plink-to.py`** — checks if the upstream is alive, primes it if not, then connects as a downstream:

```python
def shareexists(session):
    r = subprocess.run([PLINK, "-shareexists", "-batch", "-load", session],
                       capture_output=True)
    return r.returncode == 0

def start_upstream(session):
    si = subprocess.STARTUPINFO()
    si.dwFlags = subprocess.STARTF_USESHOWWINDOW
    si.wShowWindow = 0  # SW_HIDE — hidden console, proper stdin
    subprocess.Popen(
        [PLINK, "-share", "-batch", "-load", session, "sleep infinity"],
        startupinfo=si,
        creationflags=subprocess.CREATE_NEW_CONSOLE,
    )

if not shareexists(session):
    start_upstream(session)
    # poll until ready...

subprocess.run([PLINK, "-share", "-batch", "-load", session] + cmd_args)
```

The hidden-console trick (`CREATE_NEW_CONSOLE` + `SW_HIDE`) matters: `start /b` in batch gives plink a null stdin and it exits immediately. `Start-Process -WindowStyle Hidden` in PowerShell works but costs a PowerShell startup per invocation. The Python `subprocess` approach avoids both problems.

Symlink per host (Developer Mode on Windows 11 allows this without admin):

```powershell
New-Item -ItemType SymbolicLink `
  -Path   "$env:USERPROFILE\.local\bin\myserver.cmd" `
  -Target "$env:USERPROFILE\.local\bin\plink-to.cmd"
```

---

## One-time setup per host

PuTTY needs a [saved session](https://the.earth.li/~sgtatham/putty/latest/htmldoc/Chapter4.html#config-saving) with the right hostname, key file, and [sharing flags](https://the.earth.li/~sgtatham/putty/latest/htmldoc/Chapter4.html#config-ssh-sharing). It also needs the host key cached — without it, `-batch` refuses to connect.

The setup script:
1. Creates the PuTTY saved session in the registry
2. Pipes `y` to a non-batch plink invocation to accept and cache the host key once
3. Creates the symlink
4. Starts the upstream

```powershell
setup-conductor-host.ps1 -Name myserver -FQDN myserver.internal.example.com
```

After that, `myserver` from any terminal auto-primes if the upstream died, then connects. First cold call after reboot: ~8s (auth check, once). Every subsequent call: ~3s (1s plink downstream + Python + cmd startup).

The setup script is the only thing needed on a new machine — no manually noted fingerprints required. Plink does not support a `StrictHostKeyChecking=no` equivalent; the `y`-pipe approach is [the documented workaround](https://serverfault.com/a/420527).

---

## Read further

- [OpenSSH multiplexing cookbook](https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Multiplexing) — the Linux/macOS baseline this is trying to replicate
- [PROTOCOL.mux](https://github.com/openssh/openssh-portable/blob/master/PROTOCOL.mux) — OpenSSH mux wire protocol; PuTTY's sharing protocol is derived from the same ideas but uses `SSHCONNECTION@putty.projects.tartarus.org` as the version string
- [PuTTY connection sharing docs](https://the.earth.li/~sgtatham/putty/latest/htmldoc/Chapter4.html#config-ssh-sharing) — upstream/downstream model, `-shareexists`, `-share` flags
- [Win32-OpenSSH ControlMaster issue #1328](https://github.com/PowerShell/Win32-OpenSSH/issues/1328) — the open issue tracking native ControlMaster support on Windows
- [VS Code Remote SSH: ControlMaster not supported on Windows](https://github.com/microsoft/vscode-remote-release/issues/96) — same problem, different context
- [PuTTY wishlist: dedicated-sharing-upstream](https://www.chiark.greenend.org.uk/~sgtatham/putty/wishlist/dedicated-sharing-upstream.html) — the proposed `pshare` utility that would make all of this unnecessary
- [ssh/sharing.c](https://git.tartarus.org/?p=simon/putty.git;a=blob;f=ssh/sharing.c) — PuTTY sharing implementation; the `-N` hang is visible in the `CHANNEL_OPEN` handler

---

Full scripts (plink-to.py, plink-to.cmd, setup-conductor-host.ps1) in this [gist](https://gist.github.com/ankitg12/322cf69a8418ea5a3a46dae0a0eaa4f3).
