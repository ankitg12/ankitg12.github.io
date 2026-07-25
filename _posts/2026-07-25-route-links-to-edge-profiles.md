---
layout: post
title: "One Browser, Two Identities: Route Links to the Right Edge Profile"
date: 2026-07-25 21:05:00 +0530
categories: windows tools productivity
---

A link clicked outside the browser kept opening in whichever Edge profile happened to be active last — work sites in my personal session, ordinary links in my work session, and local development URLs wherever Edge felt like putting them.

The browser executable was not the problem. The missing layer was a small **link router** between Windows and the browser.

## Why the default browser setting is not enough

Windows associates `http` and `https` with an application. It does not natively express rules such as:

- work domains → Edge work profile;
- everything else → Edge personal profile;
- most localhost ports → Firefox;
- one particular localhost service → personal Edge;
- one organization under `github.com` → work Edge, while the rest of GitHub stays personal.

Edge already supports selecting a profile at launch:

```powershell
msedge.exe --profile-directory="Default" https://example.com
msedge.exe --profile-directory="Profile 1" https://example.com
```

The important detail is that `Default` and `Profile 1` are **profile directory names**, not the friendly labels shown in Edge's profile menu. A UI label such as “Personal” can still live in a directory named `Profile 1`.

Open `edge://version` in each profile and read **Profile path**. The last path component is the value required by `--profile-directory`.

## Use BrowseRouter as the dispatcher

[BrowseRouter](https://github.com/nref/BrowseRouter) is a small Windows URL dispatcher. Set it as the Windows default handler for HTTP and HTTPS, then describe several logical browsers in its `config.json` — even when those logical browsers all invoke the same executable.

```json
{
  "notify": {
    "enabled": true,
    "silent": true
  },
  "log": {
    "enabled": true
  },
  "browsers": {
    "edge-personal": "\"C:\\Program Files (x86)\\Microsoft\\Edge\\Application\\msedge.exe\" --profile-directory=\"Profile 1\"",
    "edge-work": "\"C:\\Program Files (x86)\\Microsoft\\Edge\\Application\\msedge.exe\" --profile-directory=\"Default\"",
    "firefox": "C:\\Program Files\\Mozilla Firefox\\firefox.exe"
  },
  "urls": {
    "localhost:9999": "edge-personal",
    "127.0.0.1*": "firefox",
    "localhost*": "firefox",
    "/^github[.]com/example-corp(?:/|$)/": "edge-work",
    "work.example.com": "edge-work",
    "*.work.example.com": "edge-work",
    "*": "edge-personal"
  },
  "sources": {},
  "filtersFile": "filters.json"
}
```

There are three useful patterns here.

### 1. Make the fallback explicit

```json
"*": "edge-personal"
```

Without an explicit catch-all, BrowseRouter currently adds one targeting the first configured browser. Explicit configuration is easier to review: a future reordering of the `browsers` object cannot silently change the default identity.

### 2. Put specific exceptions before broad rules

```json
"localhost:9999": "edge-personal",
"localhost*": "firefox"
```

BrowseRouter selects the first matching URL preference. The exact port exception must therefore precede the broad localhost wildcard.

There is also a source-level nuance not obvious from a casual reading of the README: simple patterns are tested against `.NET`'s `Uri.Authority`, which includes the port. An exact rule for `localhost` does **not** match `localhost:8080`. Use `localhost*` when all ports should match, then place any port-specific exceptions before it.

### 3. Use a path regex when a whole domain is too broad

Both work and personal activity may live on `github.com`. Routing the whole domain to one identity is therefore wrong. BrowseRouter's regex form evaluates the domain and path together:

```json
"/^github[.]com/example-corp(?:/|$)/": "edge-work"
```

This catches the organization root and everything beneath it without catching a similarly prefixed organization such as `example-corporation`.

## Derive rules from evidence, not memory

Browser history is already a labelled dataset: each Edge profile has its own `History` SQLite database.

Typical paths are:

```text
%LOCALAPPDATA%\Microsoft\Edge\User Data\Default\History
%LOCALAPPDATA%\Microsoft\Edge\User Data\Profile 1\History
```

Close Edge before querying these files; an active browser normally holds the database lock. Then inspect frequently visited URLs in each profile:

```powershell
sqlite3 "$env:LOCALAPPDATA\Microsoft\Edge\User Data\Default\History" @'
SELECT
  url,
  visit_count,
  datetime(last_visit_time / 1000000 - 11644473600, 'unixepoch') AS last_seen
FROM urls
ORDER BY visit_count DESC
LIMIT 100;
'@
```

The subtraction converts Chromium's microseconds-since-1601 timestamp to Unix seconds. Group the resulting URLs by hostname and compare profiles.

Treat the output conservatively:

| Observation | Routing decision |
|---|---|
| Host appears only in the work profile and carries work authentication | Work rule candidate |
| Host appears heavily in both profiles | Keep the default; consider a path regex |
| Generic identity provider or cloud login host | Usually do not route alone; let navigation inherit the originating profile |
| Old or dormant profile | Do not infer current policy from it |
| Private IP or localhost | Route explicitly; broad private-network rules often catch home services too |

History is evidence, not truth. A previously broken setup may have contaminated the work profile with personal browsing. Start with high-confidence domains, make personal the fallback, and add rules when a real misroute occurs.

## Verify the destination, not merely the command

BrowseRouter's log proves which rule matched and which command it launched:

```text
Found URL preference "*" => "edge-personal"
Launching ...msedge.exe with args "--profile-directory=\"Profile 1\" ..."
```

That is necessary but not sufficient. The stronger check is:

1. open one uniquely tagged work URL through the Windows HTTP association;
2. open one tagged personal URL the same way;
3. close Edge so both History databases are readable;
4. query each database for the tags.

```sql
SELECT url, title
FROM urls
WHERE url LIKE '%router_test=%'
ORDER BY last_visit_time DESC;
```

The tags should appear only in their intended profile database. This catches quoting mistakes, wrong profile-directory names, and launchers that log the desired command without actually producing the desired browser state.

## The broader pattern

A browser profile is an **identity and policy boundary**, not merely a different theme. It owns cookies, authentication tokens, extensions, history, password state, and enterprise policy. Link routing is therefore a small identity-control system:

```text
clicked URL
    ↓
Windows HTTP/HTTPS association
    ↓
BrowseRouter policy
    ├── work domain/path ──→ Edge --profile-directory="Default"
    ├── personal/default ──→ Edge --profile-directory="Profile 1"
    └── local development ─→ Firefox
```

Once routing is deterministic, the browser stops asking you to notice which identity happened to be active. That is exactly the sort of decision a machine should make consistently so a human no longer has to.

## Source

- [BrowseRouter repository and configuration documentation](https://github.com/nref/BrowseRouter)
- [BrowseRouter URL matching implementation](https://github.com/nref/BrowseRouter/blob/main/BrowseRouter/Config/UrlPreferenceExtensions.cs)
- [Microsoft Q&A: starting Edge with a specific profile directory](https://learn.microsoft.com/en-us/answers/questions/2372362/starting-microsoft-edge-with-specific-profile-usin)
- [Chromium user data directory documentation](https://chromium.googlesource.com/chromium/src/+/HEAD/docs/user_data_dir.md)
