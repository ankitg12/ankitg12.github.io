---
layout: post
title: "Secrets in the Keyring: What Moving to Linux Made Me Look at Again"
date: 2026-10-08
categories: linux security productivity
series: "Moving to Linux"
---

My LLM gateway key was in `.bashrc`, in plain text, as an `export`. That one line was the reason my bash dotfiles were not in my dotfiles repo.

I did not put it there on purpose. I copied it from how I had done it on Windows, and on Windows I had never thought about it again.

## Windows: two stores, used for different things

On Windows I had two places for secrets, and I used both.

**The registry, for API tokens.** The gateway key was a user environment variable, set once with [`setx`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setx). `setx` writes to `HKCU\Environment` in the registry. The value is plain text in your user hive, and every process you start inherits it. My own notes from that time say "read it via `winreg`, not the process environment".

**Credential Manager, for SSH passwords.** My SSH/console launcher pushes my public key to a lab host the first time I connect. That needs the host's password once. In August I [wrote about]({% post_url 2026-08-18-ssh-key-push-on-windows-eight-line-bug %}) the native dialog (`CredUIPromptForCredentialsW`) plus [`CredWriteW`](https://learn.microsoft.com/en-us/windows/win32/api/wincred/nf-wincred-credwritew) / `CredReadW`: prompt once, store the password under a target name such as `pcon:<host>`, read it back silently next time, and delete it when the host rejects it. Credential Manager encrypts each entry per user with [DPAPI](https://learn.microsoft.com/en-us/windows/win32/api/dpapi/nf-dpapi-cryptprotectdata), with a key derived from the login.

So I had the right tool and used it for one class of secret, but not for the long-lived API tokens. The rule I had in my head was "Credential Manager is for passwords a dialog asks for; tokens go in environment variables". That rule was about *how the secret arrives*, not about *how much damage it does if it leaks*. A migration makes you touch every habit once, and that is when you see the ones you stopped seeing.

| | Windows | Linux |
|---|---|---|
| Encrypted per-user store | Credential Manager (DPAPI) | Secret Service (GNOME Keyring, KWallet) |
| Command-line client | `cmdkey` (cannot read values back) | `secret-tool store / lookup` |
| Program API | `CredReadW` / `CredWriteW` | libsecret, or Python `keyring` / `secretstorage` |
| Unlocked by | Windows logon | login password via PAM |
| Plain-text habit I had | `setx` → `HKCU\Environment` | `export` in `~/.bashrc` |

## Linux: the Secret Service API

The Linux equivalent of Credential Manager is the [Secret Service API](https://specifications.freedesktop.org/secret-service-spec/latest/), a freedesktop.org D-Bus standard. [GNOME Keyring](https://wiki.gnome.org/Projects/GnomeKeyring) implements it, and so does KWallet. The keyring is encrypted with your login password and unlocked when you log in. [`secret-tool`](https://manpages.ubuntu.com/manpages/noble/man1/secret-tool.1.html) is the command-line client.

Store each secret once, keyed by attributes:

```bash
secret-tool store --label='LLM gateway key' service llm-gateway account me@example.com
secret-tool store --label='Jira/Confluence token' service atlassian account me@example.com
# paste the value, Enter, Ctrl-D
```

Then a loader that holds **no values**, only lookups:

```bash
# ~/.config/secrets.sh — sourced by ~/.bashrc; safe to commit
command -v secret-tool >/dev/null 2>&1 || return 0
_secret() { secret-tool lookup service "$1" account me@example.com 2>/dev/null; }

LLM_GATEWAY_KEY="$(_secret llm-gateway)"
_atl="$(_secret atlassian)"
if [ -z "$LLM_GATEWAY_KEY" ] || [ -z "$_atl" ]; then
    echo "secrets.sh: keyring lookup empty (locked?)" >&2
fi
export LLM_GATEWAY_KEY
export JIRA_API_TOKEN="$_atl"
export CONFLUENCE_TOKEN="$_atl"
unset _atl; unset -f _secret
```

And in `.bashrc`:

```bash
[ -r "$HOME/.config/secrets.sh" ] && . "$HOME/.config/secrets.sh"
```

My coding-agent harness already did this: its model config reads the key with `!secret-tool lookup service llm-gateway ...`, and the harness runs the command at startup. The shell was the last plain-text copy. The loader reads the same keyring entry, so one entry serves both, and a rotation updates both.

Now `.bashrc`, `.profile` and `secrets.sh` all go into the dotfiles repo. The values exist in one place only. One lookup takes 41–45 ms on my machine; two lookups add about 90 ms to each new shell.

Before I trusted it, I proved two things. First, the keyring value matches the old one byte for byte. Second, Jira and Confluence clients work from the new variables in a `$HOME` that has no `~/.netrc`, so nothing silently falls back to the old plain-text copy.

## Runtime lookup or template-time lookup?

chezmoi can do this for you: its [`secret` template function](https://www.chezmoi.io/reference/templates/secret-functions/secret/) calls `secret-tool` when it renders a file. I chose a runtime loader instead, for one reason. With a template, the rendered file on disk holds the plain value again, and a rotated token needs `chezmoi apply`. With a runtime loader, the file never holds a value, and rotation is one `secret-tool store`.

## What this does not fix

The keyring protects secrets **at rest**. Once the loader exports a variable, any process running as you can read it from [`/proc/<pid>/environ`](https://man7.org/linux/man-pages/man5/proc_pid_environ.5.html). That was also true of the Windows registry variable. What changed is that the secret is no longer in a file that backups, `grep`, or a dotfiles commit can pick up.

The failure mode is a **locked keyring**. The login password unlocks it ([ArchWiki: GNOME/Keyring](https://wiki.archlinux.org/title/GNOME/Keyring)). An SSH-only session, or a login that does not supply a password, can leave it locked. The loader then prints one warning line, so the problem is visible and not silent. For services and timers that must run with no one logged in, the next step is `systemd-creds`, which encrypts a credential with the TPM and gives it to one unit as a file.

## A finding along the way

Before the first commit I ran [gitleaks](https://github.com/gitleaks/gitleaks) over the dotfiles repo:

```bash
gitleaks detect --no-git --source ~/.local/share/chezmoi --redact
```

It found a GitHub OAuth token in an old MCP config file, committed months ago. I checked it with `gh api user`: `Bad credentials`. It was dead, but it had been sitting in the repo the whole time. Run the scanner before you commit, not after.

## Source

- [Secret Service API specification](https://specifications.freedesktop.org/secret-service-spec/latest/)
- [secret-tool(1)](https://manpages.ubuntu.com/manpages/noble/man1/secret-tool.1.html)
- [chezmoi `secret` template function](https://www.chezmoi.io/reference/templates/secret-functions/secret/)
- [gitleaks](https://github.com/gitleaks/gitleaks)
- [proc_pid_environ(5)](https://man7.org/linux/man-pages/man5/proc_pid_environ.5.html)
- [setx](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setx) and [CryptProtectData (DPAPI)](https://learn.microsoft.com/en-us/windows/win32/api/dpapi/nf-dpapi-cryptprotectdata)
