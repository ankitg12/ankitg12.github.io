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

The first connection pays the cost; every subsequent one is instant — same TCP connection, new SSH channel. [OpenSSH multiplexing](https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Multiplexing) is well-documented and widely used.

On Windows, this doesn't work. OpenSSH's ControlMaster is silently broken — the config parses without error, no socket is ever created, nothing multiplexes. This is a [known open issue](https://github.com/PowerShell/Win32-OpenSSH/issues/1328) in Win32-OpenSSH, also surfaced in the [VS Code Remote SSH tracker](https://github.com/microsoft/vscode-remote-release/issues/96).

---

## Dead ends

**Windows OpenSSH ControlMaster.** As above — setting `ControlMaster auto` in `~/.ssh/config` produces no effect on Windows. Silent failure, no diagnostic.

**PuTTY connection sharing with `-nc localhost:22` as ProxyCommand.** Plink holds the upstream connection; downstream `ssh` invocations proxy through it via a `direct-tcpip` channel to localhost:22. In practice: 13 seconds. The channel opens a new TCP socket on port 22 — the server sees a new connection and the auth check fires again.

**PuTTY sharing with `-N` upstream.** Run `plink -share -N ...` in the background to hold the connection open, then use `plink -share ... hostname` as the downstream. The downstream hangs indefinitely.

Reading [ssh/sharing.c](https://git.tartarus.org/?p=simon/putty.git;a=blob;f=ssh/sharing.c) explains why. With `-N`, PuTTY opens no session channel. When a downstream sends `SSH2_MSG_CHANNEL_OPEN`, the upstream forwards it to the server — but the server has no active session to respond against. No `CHANNEL_OPEN_CONFIRMATION` comes back. The downstream waits forever.

The PuTTY wishlist has a related open item from 2013: [dedicated-sharing-upstream](https://www.chiark.greenend.org.uk/~sgtatham/putty/wishlist/dedicated-sharing-upstream.html) — a proposed `pshare` utility that would act as a persistent headless upstream and handle this correctly.

---

## What works

The upstream needs a real session channel open. Change one word:

```
plink -share -batch -load <session> "sleep infinity"
```

This opens an actual session channel on the server. Downstream connections can now add further channels to the same authenticated connection — no new TCP handshake, no auth check. Each downstream call takes about 1 second.

The key insight: **the auth check gates TCP connections, not SSH channels.** ControlMaster works on Linux for the same reason — it multiplexes sessions over a single connection, never triggering a second auth.

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

On Windows, `%~n0` in a `.cmd` file gives the script's own name even through a symlink — the same trick works.

**`plink-to.cmd`** — the master shim, called as `server1`, `server2`, etc.:
```batch
@echo off
python "%~dp0plink-to.py" "%~n0" %*
```

**`plink-to.py`** — checks if the sharing upstream is alive, starts it if not, then connects as a downstream (abbreviated; [full script in gist](https://gist.github.com/ankitg12/322cf69a8418ea5a3a46dae0a0eaa4f3)):

```python
def shareexists(session):
    r = subprocess.run([PLINK, "-shareexists", "-batch", "-load", session],
                       capture_output=True)
    return r.returncode == 0

def start_upstream(session):
    si = subprocess.STARTUPINFO()
    si.dwFlags = subprocess.STARTF_USESHOWWINDOW
    si.wShowWindow = 0  # SW_HIDE
    subprocess.Popen(
        [PLINK, "-share", "-batch", "-load", session, "sleep infinity"],
        startupinfo=si,
        creationflags=subprocess.CREATE_NEW_CONSOLE,
    )

if not shareexists(session):
    start_upstream(session)
    # poll until pipe appears...

subprocess.run([PLINK, "-share", "-batch", "-load", session] + cmd_args)
```

The `CREATE_NEW_CONSOLE` + `SW_HIDE` combination matters. `start /b` in batch gives plink a null stdin and it exits immediately. `Start-Process -WindowStyle Hidden` in PowerShell works but adds ~1s per call. The Python `subprocess` approach gets a proper hidden console without the startup overhead.

Symlink per host (Developer Mode on Windows 11 allows this without admin):

```powershell
New-Item -ItemType SymbolicLink `
  -Path   "$env:USERPROFILE\.local\bin\myserver.cmd" `
  -Target "$env:USERPROFILE\.local\bin\plink-to.cmd"
```

---

## One-time setup per host

PuTTY needs a [saved session](https://the.earth.li/~sgtatham/putty/latest/htmldoc/Chapter4.html#config-saving) with the hostname, key file, and [sharing flags](https://the.earth.li/~sgtatham/putty/latest/htmldoc/Chapter4.html#config-ssh-sharing) enabled. It also needs the host key cached — without it, `-batch` refuses to connect.

A setup script handles this once:

1. Creates the PuTTY saved session in the registry
2. Pipes `y` to a non-batch plink call to accept and cache the host key
3. Creates the symlink in `~/.local/bin`
4. Starts the upstream

```powershell
setup-plink-host.ps1 -Name myserver -FQDN myserver.internal.example.com
```

After that, typing `myserver` auto-primes the upstream if it has died and then connects. First call after a reboot: ~8s (auth check runs once). Every subsequent call: ~3s (plink downstream + Python startup).

Plink has no `StrictHostKeyChecking=no` equivalent; the `y`-pipe approach is [the standard workaround](https://serverfault.com/a/420527).

---

## Read further

- [OpenSSH multiplexing cookbook](https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Multiplexing) — the Linux/macOS baseline this is trying to replicate
- [PuTTY connection sharing docs](https://the.earth.li/~sgtatham/putty/latest/htmldoc/Chapter4.html#config-ssh-sharing) — upstream/downstream model, `-share`, `-shareexists` flags
- [Win32-OpenSSH ControlMaster issue #1328](https://github.com/PowerShell/Win32-OpenSSH/issues/1328) — the open issue tracking native ControlMaster on Windows
- [PuTTY wishlist: dedicated-sharing-upstream](https://www.chiark.greenend.org.uk/~sgtatham/putty/wishlist/dedicated-sharing-upstream.html) — the `pshare` proposal that would make this unnecessary
- [ssh/sharing.c](https://git.tartarus.org/?p=simon/putty.git;a=blob;f=ssh/sharing.c) — where the `-N` hang is visible in the `CHANNEL_OPEN` handler

---

Full scripts in this [gist](https://gist.github.com/ankitg12/322cf69a8418ea5a3a46dae0a0eaa4f3).
