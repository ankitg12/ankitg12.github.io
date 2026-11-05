---
layout: post
title: "I Wanted to Delete Junk Mail. I Ended Up With Three Mail Clients."
date: 2026-11-04
categories: ai agents tools email
series: "AI coding agent productivity"
---

Gmail cleanup should take five minutes. Mine took a Himalaya install, an OAuth dead end, an IMAP app password, a TUI that doesn't exist, and an `aerc` wizard that ignored my config file. The junk mail is still there. The tooling is finally right.

## The job: one inbox, one question

One personal Gmail inbox, one day of mail. Newsletters, promos, order confirmations, bank alerts, tech mailing lists. The ask was simple: list today's mail, group the junk, delete it in bulk.

The tool on hand was `gog`, a Google CLI covering Gmail/Calendar/Drive. It worked — a JSON search returned everything. But raw JSON for dozens of mails is context arson: thousands of tokens of labels, internal dates, and thread IDs when I needed sender, subject, date.

## Step 1: the compact wrapper (build, then rename)

So I built `gmail.py`: one wrapper over the Gmail CLI with two disciplines borrowed from an existing Graph CLI with a [TTL cache](https://learn.microsoft.com/en-us/graph/) (short TTL for lists, 24 h for bodies) and progressive disclosure (one line per mail, one line per sender, full body only on demand):

```bash
python3 ~/tools/gmail.py today     # today's mail, one line each, cached
python3 ~/tools/gmail.py groups    # sender groups, biggest first
python3 ~/tools/gmail.py show 3    # full message, only the one asked
python3 ~/tools/gmail.py trash --query "..."           # dry run
python3 ~/tools/gmail.py trash --query "..." --apply   # really trash
```

First run was a live fetch. Every re-run inside TTL is instant and costs zero network. That part worked on the first try.

Then came the naming correction, and it was the right one: don't call the tool "triage". Triage is the process; the tool only supplies information.

## Step 2: the TUI question (DRW done right)

The obvious question: doesn't a Gmail TUI already exist? Yes. [Himalaya](https://github.com/pimalaya/himalaya) was one install away — Rust, IMAP/SMTP/Gmail-API backends. Check before building; the wrapper still earns its place (agents need low-context facts, not a TUI), but the human path was always Himalaya or equivalent.

## Step 3: the wizard needs human hands

`himalaya configure` is interactive and needs the human: email address, IMAP-vs-API choice, SASL mechanism, password strategy. An agent cannot finish it, and that is by design — the secret must come from human hands.

The fork that mattered, confirmed by [discussion #696](https://github.com/pimalaya/himalaya/discussions/696): a Google OAuth client secret is **not** an IMAP password. Two paths:

| Path | Needs | Speed |
|---|---|---|
| Gmail API + OAuth2 bearer via [ortie](https://github.com/pimalaya/ortie) | Cloud client, token broker | slow, more moving parts |
| IMAP + app password | 2-Step Verification + one app password | fast, done in minutes |

IMAP won. LOGIN mechanism, app password, done.

## Step 4: the password hygiene lesson

Mid-wizard, an app password landed in chat. That password was burned the moment it appeared — revoke it, generate a fresh one. Lesson restated for the record: the wizard with the OS-keychain strategy never asks you to paste the secret; it asks for the **keyring label**. Store first, point later:

```bash
security add-generic-password -a <address> -s gmail-app-password -w
# then give the wizard the label: gmail-app-password
```

The name matters too: a client-specific label was wrong the moment a second client showed up. One shared `gmail-app-password` entry now serves every IMAP client.

## Step 5: skip the wizard, write the file

The wizard's Keychain lookup failed even with the entry present. Rather than fight it, read the [config schema from the README](https://github.com/pimalaya/himalaya?tab=readme-ov-file#gmail) and write the config directly, with the password command as an argv array:

```toml
[accounts.gmail]
email = "<address>"
default = true

[accounts.gmail.imap]
server = "imaps://imap.gmail.com:993"
sasl.plain.username = "<address>"
sasl.plain.password.command = ["security", "find-generic-password",
  "-a", "<address>", "-s", "gmail-app-password", "-w"]

[accounts.gmail.smtp]
server = "smtps://smtp.gmail.com:465"
sasl.plain.username = "<address>"
sasl.plain.password.command = ["security", "find-generic-password",
  "-a", "<address>", "-s", "gmail-app-password", "-w"]
```

`himalaya account check` → `imap: OK, smtp: OK` on the first try. The credential was already proven; the file just needed to be in the documented shape.

## Step 6: no TUI in Himalaya v2

The correction worth its own section: **there is no TUI in Himalaya v2**. The old TUI repo was split out and archived. The v2 binary is a CLI with a `repl` and a wizard — the envelope table (`envelope list -m inbox -s 10 -w 200`) is a CLI table, not a TUI.

For a real terminal mail UI, the answer was [aerc](https://aerc-mail.org/), pointed at the same Keychain entry:

```ini
[personal]
from=Your Name <address>
source=imaps://<user>%40gmail.com@imap.gmail.com:993
outgoing=smtps://<user>%40gmail.com@smtp.gmail.com:465
source-cred-cmd=security find-generic-password -a <address> -s gmail-app-password -w
outgoing-cred-cmd=security find-generic-password -a <address> -s gmail-app-password -w
```

Three aerc gotchas, each a one-liner once seen: the macOS build reads `~/Library/Preferences/aerc/`, not `~/.config/aerc/`; `accounts.conf` takes only account sections (a stray `[general]` block sends aerc to the new-account wizard); and the file must be `chmod 600` or aerc refuses it as a security issue.

The folders-left/messages-right view is aerc running against Gmail IMAP. It worked on reopen after the third fix.

The human verdict at the end: "Finally!!" with a screenshot of the working TUI.

## Why not just use the Gmail API path?

OAuth2 bearer tokens expire; the IMAP app password doesn't. For a personal TUI plus agent scripting, one Keychain entry serving both clients beats a token broker that needs refreshing. If Gmail labels ever need to be first-class objects instead of IMAP mailboxes, the API path with `ortie` is still there — one block swap in the config.

## Later

The junk mail itself: still in the inbox. The dry run (`trash --query`, no `--apply`) is next, then `--apply` on human confirmation. A local indexer is installed for aerc search but not yet wired to the Gmail account. And the local CLI notes still claim work mail has no route on this machine — stale since the Graph CLI landed; one line to fix.

## Source

- [Himalaya](https://github.com/pimalaya/himalaya/)
- [Himalaya discussion #696 — Gmail setup, app password vs OAuth](https://github.com/pimalaya/himalaya/discussions/696)
- [Himalaya Gmail config docs](https://github.com/pimalaya/himalaya?tab=readme-ov-file#gmail)
- [aerc](https://aerc-mail.org/)
- [ortie](https://github.com/pimalaya/ortie)
- [Microsoft Graph](https://learn.microsoft.com/en-us/graph/) (TTL-cache pattern reference)
