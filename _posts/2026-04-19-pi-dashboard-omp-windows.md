---
layout: post
title: "Running pi-dashboard with Oh My Pi on Windows"
date: 2026-04-19
categories: ai-agents windows tooling
---

[pi-agent-dashboard](https://github.com/BlackBeltTechnology/pi-agent-dashboard) is a web UI for monitoring and interacting with [pi](https://github.com/badlogic/pi-mono) / [Oh My Pi](https://www.npmjs.com/package/@oh-my-pi/pi-coding-agent) sessions from any browser. It shows live token counts, tool call cards, costs, and lets you send prompts from a phone or second screen.

Getting it running with Oh My Pi on Windows has a few non-obvious friction points.

---

## Installation

The README says:

```bash
pi install npm:@blackbelt-technology/pi-dashboard
```

This works if you are using the original `@mariozechner/pi-coding-agent`. The package is not published to npm yet, so the command fails with a 404.

Install from the GitHub URL instead:

```bash
pi install https://github.com/BlackBeltTechnology/pi-agent-dashboard
```

`pi install` clones the repo to `~/.pi/agent/git/github.com/BlackBeltTechnology/pi-agent-dashboard`, runs `npm install`, and registers the entry in `~/.pi/agent/settings.json`.

---

## omp does not read pi's settings

`pi install` writes to `~/.pi/agent/settings.json`. Oh My Pi reads from `~/.omp/agent/config.yml`. They are separate config files. Extensions registered via `pi install` do not appear in omp sessions.

Extensions for omp go in `config.yml` under an `extensions` key:

```yaml
extensions:
  - ~/.pi/agent/git/github.com/BlackBeltTechnology/pi-agent-dashboard
```

The path is a directory; omp reads the `pi.extensions` field from `package.json` inside it to find the actual entry file.

---

## pi-dashboard binary fails outside its install directory

After registering the global binary via `npm install -g`, running `pi-dashboard` from any other directory fails:

```
Error [ERR_MODULE_NOT_FOUND]: Cannot find package 'tsx'
```

The npm-generated `.cmd` wrapper passes `--import tsx` to node. Node resolves bare specifiers in `--import` from the current working directory, not from the package. `tsx` lives in the dashboard's own `node_modules`, so it is only found when you are in that directory.

The fix is to patch `%APPDATA%\npm\pi-dashboard.cmd` to use the absolute path to tsx as a `file://` URL:

```batch
SET "TSX=%dp0%node_modules\@blackbelt-technology\pi-agent-dashboard\node_modules\tsx\dist\esm\index.mjs"
SET "CLI=%dp0%node_modules\@blackbelt-technology\pi-agent-dashboard\packages\server\src\cli.ts"
"%_prog%" --import "file:///%TSX:\=/%" "%CLI%" %*
```

The `file:///` prefix and the backslash-to-forward-slash conversion (`%TSX:\=/%`) are both required — Node's ESM loader rejects Windows paths without them.

After this, `pi-dashboard status`, `pi-dashboard start`, and `pi-dashboard stop` work from any directory.

---

## Starting the server

The built-in daemon mode (`pi-dashboard start`) has a reliability issue on Windows: it uses `process.env.HOME` to locate the log directory, which is not set by default. The spawned process exits before creating the log file, the parent times out after 5 seconds, and there is no diagnostic.

Run it as a hidden background process via PowerShell instead:

```powershell
$dir = "$env:USERPROFILE\.pi\agent\git\github.com\BlackBeltTechnology\pi-agent-dashboard"
$log = "$env:USERPROFILE\.pi\dashboard"
New-Item -ItemType Directory -Force $log | Out-Null

Start-Process node `
    -ArgumentList "--import", "tsx/esm", "packages/server/src/cli.ts" `
    -WorkingDirectory $dir `
    -WindowStyle Hidden `
    -RedirectStandardOutput "$log\server.log" `
    -RedirectStandardError  "$log\server.err"
```

The server listens on `:8000` (browser) and `:9999` (bridge WebSocket). Open `http://localhost:8000` — historical sessions appear immediately from the JSONL files on disk.

---

## Bridge extension fails under omp: missing @mariozechner/pi-ai

The dashboard's bridge extension (`packages/extension/src/bridge.ts`) connects a running session to the server over port 9999, enabling live streaming. It imports `@mariozechner/pi-ai` at runtime. Oh My Pi ships `@oh-my-pi/pi-ai` instead. Bun's module resolver does not find the mariozechner package and the extension fails to load.

Fix with directory junctions inside the dashboard's `node_modules`:

```powershell
$dash  = "$env:USERPROFILE\.pi\agent\git\github.com\BlackBeltTechnology\pi-agent-dashboard\node_modules"
$omp   = "$env:USERPROFILE\scoop\persist\bun\install\global\node_modules\@oh-my-pi"
$link  = "$dash\@mariozechner"
New-Item -ItemType Directory -Force $link | Out-Null

foreach ($pkg in "pi-ai", "pi-coding-agent", "pi-tui") {
    New-Item -ItemType Junction -Path "$link\$pkg" -Target "$omp\$pkg" | Out-Null
}
```

With the junctions in place, the bridge resolves `@mariozechner/pi-ai` → `@oh-my-pi/pi-ai` and loads successfully. Port 9999 gets an established connection from the omp session.

Note: the bridge adds latency to the session input loop. Whether this is acceptable depends on your use case. The dashboard shows historical sessions from disk without the bridge — disable it and only enable for targeted live monitoring.

---

## Windows Firewall

On first run, Windows Firewall prompts to allow Bun network access. Localhost traffic bypasses the firewall regardless, so click Cancel. If you want a precise rule:

```powershell
New-NetFirewallRule -DisplayName "pi-dashboard (localhost)" `
    -Direction Inbound -Protocol TCP -LocalPort 8000,9999 `
    -LocalAddress 127.0.0.1 -RemoteAddress 127.0.0.1 `
    -Action Allow -Profile Any
```

---

## Summary

| Problem | Fix |
|---|---|
| `npm:` package not found | Install via `https://` GitHub URL |
| omp ignores pi's settings | Add extension path to `~/.omp/agent/config.yml` |
| `pi-dashboard` fails outside its dir | Patch `.cmd` to use `file://` URL for tsx loader |
| Daemon mode silently fails on Windows | Use `Start-Process` with `-WindowStyle Hidden` |
| Bridge fails: missing `@mariozechner/pi-ai` | Junction `@mariozechner/*` → `@oh-my-pi/*` in dashboard's `node_modules` |
