---
layout: post
title: "The Eight-Line Bug at the Bottom of a $40 Detour"
date: 2026-08-18
categories: ai agents productivity windows
series: "AI coding agent productivity"
---

On Linux, "when I SSH somewhere, push my key first if it isn't there yet" is a five-minute
shell function. I wrote mine years ago with one Google search and one Stack Overflow tab:

```bash
ssh_with_key() {
    local user_host="$1"
    local ssh_key="${2:-$HOME/.ssh/id_rsa.pub}"

    ssh -o PasswordAuthentication=no "$user_host" 2>/dev/null

    if [ $? -ne 0 ]; then
        echo "Key not found on remote host. Copying key..."
        ssh-copy-id -i "$ssh_key" "$user_host"
        ssh "$user_host"
    fi
}
alias ssh='ssh_with_key'
```

The same requirement on Windows, driven through a coding agent, cost me two sessions and
roughly $40 before it worked. The actual defect at the bottom of it was three lines long.

This is the post-mortem, including the part where the agent decided to type my public key
over a **serial console**, character by character, and I did not notice for a session and a half.

## Why the Linux version is cheating

The five-minute solution is five minutes because `ssh-copy-id` exists and the operating
system hands you a TTY. Neither is true on Windows when your terminal is PuTTY.

| Assumption on Linux | Reality on Windows/PuTTY |
|---|---|
| `ssh-copy-id` ships with the OS | Doesn't exist; no equivalent |
| `ssh` inherits your terminal, so the password prompt just works | PuTTY is a detached GUI process; you cannot write to its stdin |
| One process asks, one process connects | The password must reach a *different* process than the one you're typing into |
| `$?` distinguishes "no key" from "host down" | You have to parse stderr to tell them apart |

So on Windows you need three things Linux gave you free: a way to *ask* for a password, a
way to *carry* it to the connecting process, and a way to *install* the key yourself. That is
genuinely more work. It is not $40 of more work.

## Detour one: the key push over a serial console

Midway through the second session I read the code properly and found that the agent had
implemented key installation by **driving a serial console** — waking a console line,
matching a shell prompt with expect-style regexes, and echoing a 400-character public key
line into `authorized_keys` over a link with no flow control.

Take a second to appreciate how bad this is. A serial console is slow, lossy, has no
integrity check, and verifies by reading back its own echo. One dropped character in a
base64 key and you have a corrupt `authorized_keys` entry that fails silently, forever, with
no error anywhere. It is the single worst transport available on the machine for moving an
exact string.

My first reaction was "where on earth did that idea come from?" My second reaction, after
grepping, was worse: **it came from my own config file.** The host entry said:

```yaml
key_recovery_via: <other-host>
# Firmware installs replace /root; recover through serial, never a stored password.
```

A previous session had reasoned: firmware upgrades wipe the home directory, therefore there
is no stable password, therefore the only recovery path is the console. It wrote that
conclusion into the config as a directive. The next session read the directive and
implemented it faithfully.

The agent was not hallucinating. It was obeying me — a stale, six-weeks-ago me.

**Lesson: your config files are part of the prompt.** Anything declarative sitting in a
YAML file your tooling reads will be treated as a requirement, not a suggestion, and it will
outlive the reasoning that produced it. When an agent goes somewhere bizarre, grep your own
repo before blaming the model. Roughly half the time the instruction is sitting right there
in your own handwriting.

## Detour two: the happy path passed

After I corrected the direction, the agent rewrote it properly — Paramiko for the SSH
connection, a dialog for the password, append the key with a `grep -qxF` guard so it stays
idempotent. The positive case worked. I tried it, the key installed, PuTTY opened.

Then I typed the wrong password on purpose. It failed and **never asked me again.** No
retry, no second dialog, nothing. Session over.

## The actual bug

Here is the entire defect:

```python
except (OSError, paramiko.SSHException) as exc:
    print(f"password bootstrap failed: {exc}", file=sys.stderr)
    return 1
```

`paramiko.AuthenticationException` is a **subclass** of `paramiko.SSHException`:

```text
SSHException
├── AuthenticationException      ← wrong password: retryable, ask the user again
│   ├── PasswordRequiredException
│   └── BadAuthenticationType
├── BadHostKeyException          ← not retryable
└── ProxyCommandFailure          ← not retryable
```

That single `except` clause flattened "the server rejected your password" and "the host is
unreachable" into the same return code. The caller received `1` and had no way to know it was
looking at a mistyped password rather than a dead network, so it did the only safe thing:
gave up. There was no retry loop to write, because there was nothing to branch on.

The same conflation existed one level up, in the reachability probe:

```python
def _key_auth_ok(target) -> bool:
    return subprocess.run([...]).returncode == 0     # False for *every* failure
```

`False` meant both "the server refused your key" (worth asking for a password) and "the host
is down" (asking for a password is pointless). So an unreachable host popped a credential
dialog that could not possibly succeed.

**Both bugs are the same bug: a lossy error channel.** Everything downstream was starved of
the one bit it needed.

## The fix

Give the failure a vocabulary. A tri-state probe:

```python
def _key_auth_state(target: str) -> str:
    """Classify key-only auth: 'ok', 'refused' (server said no), or 'unreachable'."""
    probe = subprocess.run(
        ["ssh", *SSH_EXEC_OPTIONS, "-o", "PasswordAuthentication=no", target, "true"],
        stdin=subprocess.DEVNULL, capture_output=True, text=True,
    )
    if probe.returncode == 0:
        return "ok"
    stderr = probe.stderr.lower()
    if "permission denied" in stderr or "no supported authentication" in stderr:
        return "refused"
    return "unreachable"
```

And a retryable return code, ordered so the subclass is caught first:

```python
except paramiko.AuthenticationException as exc:
    print(f"{user}@{host}: password rejected ({exc})", file=sys.stderr)
    return AUTH_REJECTED                     # 2 — ask again
except (OSError, paramiko.SSHException) as exc:
    print(f"password bootstrap failed: {exc}", file=sys.stderr)
    return 1                                 # transport failure — stop
```

Now the retry loop has something to say, and it retries *only* the rejection. `unreachable`
fails fast without ever raising a dialog. Total change: about eight lines.

## Was the password dialog even the right one?

Once it worked I asked the obvious follow-up: the prompt was a `tkinter` dialog. Is there a
PuTTY-native way to do this?

**No.** `plink` offers exactly two password options and neither of them helps:

```text
  -pw passw      login with specified password
  -pwfile file   login with password read from specified file
```

Both *consume* a password you already hold. Neither prompts and returns one, and PuTTY has
no concept of installing a key on a remote host at all. The prompt has to come from you.

`tkinter` is a legitimate choice — and it is not slow; the import measured 0.05 s, so that
common objection is a myth. Its real problems are that the dialog looks foreign on Windows
and, more importantly, that it has no memory. On a box whose home directory is wiped by every
firmware upgrade, that means retyping the password every single time.

The native answer is `credui` plus the Windows Credential Manager, both reachable from
`ctypes` with no dependencies:

| Approach | Prompt looks native | Remembers the password |
|---|---|---|
| `tkinter.simpledialog` | no | no |
| PowerShell `Get-Credential` | yes | no |
| `CredUIPromptForCredentialsW` | yes | via checkbox |
| `+ CredWriteW` / `CredReadW` | yes | **yes, encrypted per-user by Windows** |

The last row is the one that changes daily life: prompt once ever, then reinstall the key
silently on every subsequent wipe. The secret never enters the repository, a config file, or
a process argument list — which was exactly the fear that sent the first session down the
serial-console road in the first place.

The struct binding is the only fiddly part, and it is checkable in one command:

```python
>>> ctypes.sizeof(wincred._CREDENTIALW)
80
>>> wincred.write(t, 'root', 'p@ss-w0rd-é中'); wincred.read(t)
'p@ss-w0rd-é中'
```

If the layout is wrong you get garbage or a crash immediately, so a round-trip with a
non-ASCII password is a complete test of it.

## One more bug, found while replacing that code

```python
try:
    import tkinter as tk
    from tkinter import simpledialog
    ...
except (ImportError, tk.TclError):
    return getpass.getpass(...)
```

If `import tkinter` raises `ImportError`, the name `tk` was never bound — so evaluating the
`except` tuple raises `UnboundLocalError`. The fallback is unreachable in precisely the case
it was written for:

```text
LATENT BUG reproduced -> UnboundLocalError cannot access local variable 'tk'
```


## What I'd actually do differently

Three rules, all cheap, all of which would have saved most of that $40:

1. **Grep your own config before believing the agent invented something.** The most
   expensive detour of the whole exercise was a stale YAML comment being obeyed correctly.
   Declarative state is instruction, and it does not expire on its own.
2. **Ask what the failure path returns, not whether the happy path works.** For anything
   involving authentication, the happy path is the *boring* half. "It installed the key"
   told me nothing. "It returns the same code for a wrong password and a dead host" told me
   everything, and I could have asked it on minute one.
3. **Make the agent show you the error taxonomy.** One question — "what distinct failures
   can this raise, and what does the caller see for each?" — surfaces a flattened error
   channel instantly. It is the highest-yield question I know for auth, retry, and
   network code, and it is exactly the question neither session asked.

The model was never the problem here. A stale config line pointed it at a terrible design,
and a lossy `except` clause hid the real defect behind a passing happy path. Both are things
I own, and both are cheap to check the next time — which is the only part of a $40 lesson
worth keeping.

## Source

- [`ssh-copy-id(1)`](https://man7.org/linux/man-pages/man1/ssh-copy-id.1.html) — the primitive Windows lacks
- [Paramiko exception hierarchy](https://docs.paramiko.org/en/latest/api/ssh_exception.html) — `AuthenticationException` subclasses `SSHException`
- [PuTTY `plink` command line](https://the.earth.li/~sgtatham/putty/0.83/htmldoc/Chapter7.html) — `-pw` and `-pwfile`
- [`CredUIPromptForCredentialsW`](https://learn.microsoft.com/en-us/windows/win32/api/wincred/nf-wincred-creduipromptforcredentialsw) — the native dialog
- [`CredWriteW`](https://learn.microsoft.com/en-us/windows/win32/api/wincred/nf-wincred-credwritew) / [`CredReadW`](https://learn.microsoft.com/en-us/windows/win32/api/wincred/nf-wincred-credreadw) — the credential store
