---
layout: post
title: "The Thumbnail Sandbox Wasn't Working. Files Showed Icons Instead."
date: 2026-11-19
categories: linux debugging security
series: "Moving to Linux"
---

My screenshots appeared as identical purple icons in [Files](https://apps.gnome.org/Nautilus/) on [Ubuntu 24.04](https://releases.ubuntu.com/24.04/). Files could not generate their previews. The process that creates the thumbnails could not start its sandbox.

This is one failure mode, not a universal fix for missing thumbnails. Start with the evidence on your own machine, especially before changing a security policy.

## The path from an image to an icon

Files, also known as Nautilus, uses a thumbnailing component to make small previews. The desktop has a registered [GdkPixbuf](https://docs.gtk.org/gdk-pixbuf/) thumbnailer for PNG and other image formats. Generated thumbnails are stored in a user cache, according to the [thumbnail specification](https://specifications.freedesktop.org/thumbnail-spec/latest/).

The thumbnailer runs inside a sandbox built with [bubblewrap](https://github.com/containers/bubblewrap) (`bwrap`). A sandbox limits what the image-processing process can see and do. That matters because an image file is input to a parser; an image that looks harmless in a directory is still data that code must decode.

The path was:

```text
Files → thumbnail factory → bubblewrap → image thumbnailer → thumbnail cache
                                  ↑
                           AppArmor policy
```

The thumbnailer did not get a usable sandbox, so Files fell back to a generic image icon. A setting that enables previews cannot make up for a failed sandbox.

## First, check the settings

On this machine, thumbnailing was already enabled for local files:

```text
$ gsettings get org.gnome.nautilus.preferences show-image-thumbnails
'local-only'

$ gsettings get org.gnome.desktop.thumbnailers disable-all
false
```

`local-only` was appropriate here: the screenshots were on a local filesystem. A network location can need a different setting; check that separately. Neither output proves that thumbnail generation succeeds.

## The security boundary that failed

[AppArmor](https://documentation.ubuntu.com/security/security-features/privilege-restriction/apparmor/) is a Linux security module that confines programs with policy profiles. Ubuntu can use AppArmor to [restrict unprivileged user namespaces](https://ubuntu.com/blog/ubuntu-23-10-restricted-unprivileged-user-namespaces). A [user namespace](https://man7.org/linux/man-pages/man7/user_namespaces.7.html) lets a process have a distinct view of user IDs and capabilities inside that namespace; those permissions are not automatically root privileges on the host.

The trade-off is subtle. Namespaces help create sandboxes without running the application as host root, but they can also expose kernel interfaces to unprivileged processes. Ubuntu's restriction allows policy to decide which programs may use those interfaces instead of disabling the protection for the entire machine.

The kernel journal showed AppArmor denying capabilities to `bwrap` under the `unprivileged_userns` profile. This is an excerpt from one denial:

```text
apparmor="DENIED" operation="capable" class="cap" profile="unprivileged_userns" comm="bwrap" capability=8 capname="setpcap"
```

`setpcap` names a [Linux capability](https://man7.org/linux/man-pages/man7/capabilities.7.html); it is not the name of the image file or a Nautilus preference. The sandbox test also failed with:

```text
bwrap: setting up uid map: Permission denied
```

The denial and the failed sandbox test were consistent with a policy problem. They did not, on their own, prove every missing thumbnail had that cause. The confirmation was the change in behavior after the policy was loaded.

To look for the same evidence, inspect the kernel journal:

```bash
journalctl -k -b --no-pager -g 'apparmor="DENIED"'
```

Read the matching entries: denials from another program or an unrelated time do not establish this diagnosis. On managed systems, use the approved administrative process before changing AppArmor policy.

## The fix: a profile, not a system-wide switch

The installed `apparmor-profiles` package provides an upstream [restricted bubblewrap profile](https://gitlab.com/apparmor/apparmor/-/blob/master/profiles/apparmor/profiles/extras/bwrap-userns-restrict). Its comments describe the boundary: allow `bwrap` to use user namespaces and capabilities within its namespace while denying capabilities to its children. The profile is disabled by default and its own comments warn it can break some use cases.

I installed that package, linked its supplied profile into `/etc/apparmor.d/`, and reloaded AppArmor. The relevant steps were:

```bash
sudo apt install apparmor-profiles
sudo ln -s /usr/share/apparmor/extra-profiles/bwrap-userns-restrict /etc/apparmor.d/bwrap-userns-restrict
```

Before linking, check that the supplied profile exists and that the destination is not already a different profile or symlink. Read the profile and get authorization for the policy change. If the destination already exists, do not overwrite it blindly.

In this environment the reload happened through an AppArmor package reinstall, which also changed packages. The distribution's [apparmor_parser documentation](https://manpages.ubuntu.com/manpages/noble/en/man8/apparmor_parser.8.html) describes loading a profile directly; I did not test that direct reload here. Use the approved reload method for your machine and distribution.

I did **not** disable the user-namespace restriction globally. After the change, the restriction remained enabled:

```text
$ sysctl kernel.apparmor_restrict_unprivileged_userns
kernel.apparmor_restrict_unprivileged_userns = 1
```

That does not make the profile risk-free: it changes what `bwrap` is permitted to do. Review it as a security-policy change, not as a cosmetic preference.

## Verify the mechanism and the result

After the profile was loaded, this sandbox test succeeded:

```text
$ bwrap --ro-bind /usr /usr --ro-bind /lib /lib --ro-bind /lib64 /lib64 --proc /proc --dev /dev /usr/bin/true && echo BWRAP_OK
BWRAP_OK
```

The library bindings matter on this machine: without the dynamic loader, a sandbox can fail to execute `true` even when namespace creation works. The test proves that this particular sandbox invocation can start; it does not exercise the image decoder.

The end-to-end check was visual: after the thumbnail cache was refreshed and Files restarted, the screenshot folder showed image previews instead of identical icons. That is the useful distinction between a successful sandbox smoke test and a successful thumbnail workflow.

The lesson: when the preference is already correct, follow the execution chain. Here a desktop rendering symptom was caused by a security boundary several layers below the file manager.

## Source

- [Ubuntu: restricted unprivileged user namespaces](https://ubuntu.com/blog/ubuntu-23-10-restricted-unprivileged-user-namespaces)
- [AppArmor: restricted bubblewrap profile](https://gitlab.com/apparmor/apparmor/-/blob/master/profiles/apparmor/profiles/extras/bwrap-userns-restrict)
- [bubblewrap project](https://github.com/containers/bubblewrap)
- [Freedesktop thumbnail specification](https://specifications.freedesktop.org/thumbnail-spec/latest/)
