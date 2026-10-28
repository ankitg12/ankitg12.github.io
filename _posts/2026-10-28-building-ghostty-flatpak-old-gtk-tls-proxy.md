---
layout: post
title: "Building a Too-New GTK App as a User Flatpak: Five Failures, and the One My Agent Caused"
date: 2026-10-28 16:05:00 +0530
categories: ai tools linux
series: "AI coding agent productivity"
---

[Ghostty](https://ghostty.org) [1.4.0](https://ghostty.org/docs/install/release-notes/1-4-0) needs [GTK](https://www.gtk.org) 4.18. My [Ubuntu 24.04](https://releases.ubuntu.com/24.04/) laptop ships GTK 4.14, the [snap](https://snapcraft.io/ghostty) bundles 4.14 too, and [Nix](https://nixos.org) was unusable without root. The [release notes](https://ghostty.org/docs/install/release-notes/1-4-0) were hours old, so I asked my coding agent to get it running without touching the system.

The build failed five different ways, and then the agent caused a sixth problem itself. This is the log of all of them.

## The route: use the app's own Flatpak manifest

A [Flatpak](https://flatpak.org) build brings its own runtime, so the host GTK stops mattering. Ghostty ships [`flatpak/com.mitchellh.ghostty.yml`](https://github.com/ghostty-org/ghostty/tree/v1.4.0/flatpak), which targets the GNOME 50 runtime (GTK 4.22) from [Flathub](https://flathub.org). Everything installs at user scope, so no root and no change to the snap:

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

The build ran for a long time, then stopped with a [Zig](https://ziglang.org) version error. `build.zig.zon` required 0.17.0 and `flatpak/dependencies.yml` still downloaded 0.16.0. I did not edit the upstream files. A generated overlay manifest pointed at the new tarball, with the checksum taken from `ziglang.org/download/index.json`. After that the build finished, and `+version` reported 1.4.0 on GTK 4.22.5 and [libadwaita](https://gnome.pages.gitlab.gnome.org/libadwaita/) 1.9.4.

All five fixes are one script, so the next release is one command: `build.sh [checkout]`.

## The mistake that was not in the build

To smoke-test the new build, the agent used the app's GTK action interface over [D-Bus](https://dbus.freedesktop.org/doc/dbus-specification.html): open a tab, then close the window. Both the snap and the Flatpak own the name `com.mitchellh.ghostty`. The calls went to the snap, and it closed my live terminal window. The 1.4.0 instance it meant to test was untouched.

The lesson is general. **Two builds with one application ID share one bus name, and a control call reaches whoever owns it.** Before any D-Bus or single-instance call, ask the bus who answers:

```bash
busctl --user list | awk '/ghostty/ {print $1, $2}'
```

For a test instance, start it with its own `--class` and `--gtk-single-instance=false`, and send commands to that instance only.

## What a Flatpak terminal does to your launch scripts

Ghostty in a Flatpak runs your shell on the host through the `org.freedesktop.Flatpak.Development.HostCommand` D-Bus call. I tested my usual window-launch pattern (`systemd-run --user ... ghostty -e <command>`) against it:

| `-e` command | Result |
|---|---|
| `/bin/sh -c '...'` | runs on the host |
| [`tmux`](https://github.com/tmux/tmux/wiki) `new-session ...` | runs, attaches |
| `ssh` with the host agent | works |
| script under `$HOME` | runs |
| script in `/tmp` | `FlatpakSpawnFail`: "terminal is non-functional" |

The sandbox has a private `/tmp`. The executable path is resolved there, so a script that exists only in the host's `/tmp` fails with `AccessDenied`. Keep launch scripts under `$HOME`.

## Use it on your own machine

You need a Linux host with `flatpak` and `flatpak-builder` (the `org.flatpak.Builder` Flatpak works if the distro package is old), about 6 GB of free disk, and a checkout of the Ghostty tag you want.

```bash
# 1. Source at the release tag
git clone --depth 1 --branch v1.4.0 https://github.com/ghostty-org/ghostty
cd ghostty

# 2. Runtimes, user scope (about 1.3 GB)
flatpak remote-add --user --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
flatpak install --user -y flathub org.flatpak.Builder org.gnome.Platform//50 org.gnome.Sdk//50

# 3. Build and install
flatpak run --command=flatpak-builder org.flatpak.Builder \
  --user --install --force-clean --disable-rofiles-fuse \
  --state-dir=$HOME/.cache/ghostty-fpbuild/state \
  $HOME/.cache/ghostty-fpbuild/build flatpak/com.mitchellh.ghostty.yml

# 4. Run, and check the version
flatpak run com.mitchellh.ghostty +version
```

On a plain network that is all. Add the fixes above only when you see the matching error:

| Symptom | Add to the `flatpak run --command=flatpak-builder` line |
|---|---|
| TLS verification failure on downloads | `--env=CURL_CA_BUNDLE=/run/host/etc/ssl/certs/ca-certificates.crt --env=SSL_CERT_FILE=/run/host/etc/ssl/certs/ca-certificates.crt` |
| `publickey` error on an `https://` source | `--env=GIT_CONFIG_GLOBAL=<file without the insteadOf rule>` |
| `unable to read tree` | delete `state/cache` and `state/build`, rerun |
| a git source fails on a pinned SHA | fetch the SHA into `state/git/<mirror>` first, then `update-ref` |
| Zig version error | compare `minimum_zig_version` in `build.zig.zon` with the URL in `flatpak/dependencies.yml`; use an overlay manifest |

The Flatpak reads its config from `~/.var/app/com.mitchellh.ghostty/config/ghostty/config.ghostty`, not `~/.config/ghostty/`. A symlink between the two keeps one file:

```bash
mkdir -p ~/.var/app/com.mitchellh.ghostty/config/ghostty
ln -sfn ~/.config/ghostty/config.ghostty ~/.var/app/com.mitchellh.ghostty/config/ghostty/config.ghostty
```

If you also run another Ghostty build, start the Flatpak with its own class and no single-instance, as described above, so the two do not share one bus name.

## Make it the default, with a way back

Two symlinks and a wrapper do it. The wrapper runs the Flatpak, and runs the snap when `GHOSTTY_USE_SNAP` is set or the Flatpak is missing:

```sh
#!/bin/sh
if [ -n "${GHOSTTY_USE_SNAP:-}" ] || ! flatpak info --user com.mitchellh.ghostty >/dev/null 2>&1; then
  exec /snap/bin/ghostty "$@"
fi
exec flatpak run com.mitchellh.ghostty "$@"
```

```bash
ln -s ~/tools/ghostty-flatpak/ghostty ~/.local/bin/ghostty   # bare `ghostty` is now the Flatpak
```

`~/.local/bin` comes before `/snap/bin` on `PATH` here, so the link wins. To go back, delete the link. A launch script that calls bare `ghostty` (a window manager, a terminal multiplexer wrapper) picks up the change with no edit. I ran one such launch, built by my own window-launcher code, through the link: the child ran on the host with the ssh agent and the host `tmux` visible.

## Not checked

- Whether shell integration works in the Flatpak. The integration files are present inside the sandbox; I did not run a session to see them load.
- A visual check of the window. [GNOME](https://www.gnome.org) on [Wayland](https://wayland.freedesktop.org) blocks screenshots from plain clients, so the evidence is `+version`, config loading, and the D-Bus window objects.
- Anything past the launch patterns in the table above, such as a long ssh or tmux session.

## Source

- [Ghostty 1.4.0 release notes](https://ghostty.org/docs/install/release-notes/1-4-0)
- [Ghostty source](https://github.com/ghostty-org/ghostty)
- [flatpak-builder documentation](https://docs.flatpak.org/en/latest/flatpak-builder.html)
- [Flatpak sandbox permissions](https://docs.flatpak.org/en/latest/sandbox-permissions.html)
