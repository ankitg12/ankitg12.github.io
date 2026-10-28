---
layout: post
title: "Building a Too-New GTK App as a User Flatpak: Four Failures, and the One My Agent Caused"
date: 2026-10-28 16:05:00 +0530
categories: ai tools linux
series: "AI coding agent productivity"
---

Ghostty 1.4.0 needs GTK 4.18. My Ubuntu 24.04 laptop ships GTK 4.14, the snap bundles 4.14 too, and Nix was unusable without root. The release notes were hours old and I wanted the build anyway, so I asked my coding agent to get it running without touching the system.

It took one `flatpak-builder` run that failed four different ways, and then a fifth problem the agent caused itself. This is the log of all of them.

## The route: use the app's own Flatpak manifest

A Flatpak build brings its own runtime, so the host GTK stops mattering. Ghostty ships `flatpak/com.mitchellh.ghostty.yml`, which targets the GNOME 50 runtime (GTK 4.22). Everything installs at user scope, so no root and no change to the snap:

```bash
flatpak remote-add --user --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
flatpak install --user -y flathub org.flatpak.Builder org.gnome.Platform//50 org.gnome.Sdk//50
```

That is about 1.3 GB. On a slow link it is an hour of waiting, so run it as a named background job and do something else.

## Failure 1: the sandbox does not trust your network's CA

Every source download failed TLS verification. The network re-signs HTTPS traffic with a corporate CA. The host trusts it; the Builder sandbox does not. `curl` inside the sandbox showed the cause in one line: with the host bundle passed in, the same URL returned 200.

```bash
flatpak run --command=sh org.flatpak.Builder -c \
  'curl -sI -o /dev/null -w "%{http_code}\n" \
   --cacert /run/host/etc/ssl/certs/ca-certificates.crt https://ziglang.org/'
```

Fix: pass the host bundle through `--env=CURL_CA_BUNDLE=...` and `--env=SSL_CERT_FILE=...`.

## Failure 2: `~/.gitconfig` rewrites https to ssh

Git sources then failed with a publickey error, although the manifest uses `https://`. A global `url."git@github.com:".insteadOf` rule rewrote them, and the sandbox cannot read `~/.ssh/config`. Fix: point `GIT_CONFIG_GLOBAL` at a small private file that holds only the CA setting.

## Failure 3: stale builder state

After failures 1 and 2 the next run died with `unable to read tree`. The checkout and the clone were both fine; the builder's own cache held half-written objects. Deleting `state/cache` and `state/build` fixed it and kept the downloads. When a failure points at a repo that checks out fine, suspect the tool's cache before the repo.

## Failure 4: a pinned commit on no branch

Then `zigimg` failed: the manifest pins a commit that is on no branch or tag. `git fetch` of a branch never brings it in, but fetching by SHA does. Pre-fetch it into the builder's git mirror and anchor it with a ref so `gc` keeps it:

```bash
git -C "$MIRROR" fetch "$URL" "$SHA"
git -C "$MIRROR" update-ref "refs/heads/pin-${SHA:0:7}" "$SHA"
```

## Failure 5: the manifest was one Zig release behind

The build ran for a long time, then stopped with a Zig version error. `build.zig.zon` required 0.17.0 and `flatpak/dependencies.yml` still downloaded 0.16.0. I did not edit the upstream files. A generated overlay manifest pointed at the new tarball, with the checksum taken from `ziglang.org/download/index.json`. After that the build finished, and `+version` reported 1.4.0 on GTK 4.22.5 and libadwaita 1.9.4.

All five fixes are one script, so the next release is one command: `build.sh [checkout]`.

## The mistake that was not in the build

To smoke-test the new build, the agent used the app's GTK action interface over D-Bus: open a tab, then close the window. Both the snap and the Flatpak own the name `com.mitchellh.ghostty`. The calls went to the snap, and it closed my live terminal window. The 1.4.0 instance it meant to test was untouched.

The window was not carrying anything I needed, so the cost was small. The lesson is general. **Two builds with one application ID share one bus name, and a control call reaches whoever owns it.** Before any D-Bus or single-instance call, ask the bus who answers:

```bash
busctl --user list | awk '/ghostty/ {print $1, $2}'
```

For a test instance, start it with its own `--class` and `--gtk-single-instance=false`, and send commands to that instance only.

## What a Flatpak terminal does to your launch scripts

Ghostty in a Flatpak runs your shell on the host through the `org.freedesktop.Flatpak.Development.HostCommand` D-Bus call. I tested my usual window-launch pattern (`systemd-run --user ... ghostty -e <command>`) against it:

| `-e` command | Result |
|---|---|
| `/bin/sh -c '...'` | runs on the host |
| `tmux new-session ...` | runs, attaches |
| `ssh` with the host agent | works |
| script under `$HOME` | runs |
| script in `/tmp` | `FlatpakSpawnFail`: "terminal is non-functional" |

The sandbox has a private `/tmp`. The executable path is resolved there, so a script that exists only in the host's `/tmp` fails with `AccessDenied`. Keep launch scripts under `$HOME`.

## Not verified

Shell integration did not load in the Flatpak (`shell could not be detected` in the log). I did not take a screenshot, because GNOME on Wayland blocks it for plain clients. I have not yet made the Flatpak the default; the snap stays the launcher until I have used 1.4.0 for a few days.

## Source

- [Ghostty 1.4.0 release notes](https://ghostty.org/docs/install/release-notes/1-4-0)
- [Ghostty source](https://github.com/ghostty-org/ghostty)
- [flatpak-builder documentation](https://docs.flatpak.org/en/latest/flatpak-builder.html)
- [Flatpak sandbox permissions](https://docs.flatpak.org/en/latest/sandbox-permissions.html)
