---
layout: post
title: "When PuTTY's Saved Session List Stops Scaling"
date: 2026-08-17
categories: windows tools productivity
---

PuTTY's saved-session dialog becomes difficult to use long before the underlying sessions become difficult to manage: every entry appears in one flat, alphabetically sorted list.

Deleting old sessions fixes the list but loses configuration. Prefixing names creates visual groups but changes the names consumed by shortcuts and automation. Maintaining a second session database creates two sources of truth.

The useful distinction is between **session configuration** and **session presentation**. PuTTY can remain the configuration store while [PuTTY Session Manager](https://puttysm.sourceforge.io/) provides the tree view.

## The storage model

PuTTY stores saved sessions below this per-user Registry key:

```text
HKCU\Software\SimonTatham\PuTTY\Sessions
```

Each child key is one saved session. PuTTY Session Manager, or PSM, reads the same keys and adds one string value:

```text
PsmPath = Sessions\90 - Archived\Lab
```

That value is presentation metadata. It does not rename the session key and does not change its host, port, protocol, key, logging, or terminal settings.

This matters when another command launches a session by its exact saved name:

```text
putty.exe -load "lab-console"
```

Moving `lab-console` into a PSM folder leaves `-load "lab-console"` valid.

## What PSM does—and does not do

| Surface | Result |
|---|---|
| PSM tree | Sessions appear in nested folders derived from `PsmPath` |
| PuTTY native dialog | Sessions remain in the original flat list |
| `putty.exe -load` | Existing saved-session names continue to work |
| Registry | Original session values remain intact; `PsmPath` is added |

PSM is therefore an **alternative visual launcher**, not a folder extension for PuTTY's native dialog. If the requirement is to shorten the native list, sessions must be exported and removed, then restored before use. If the requirement is faster visual selection without losing anything, use the PSM tree as the launcher.

## Keep the folder map in a file

A GUI-only folder arrangement is difficult to review, reproduce, or repair. A small JSON file makes the desired tree explicit:

```json
{
  "build-1": "Sessions\\00 - Active\\Build Servers",
  "build-2": "Sessions\\00 - Active\\Build Servers",
  "lab-console": "Sessions\\10 - Lab",
  "old-router": "Sessions\\90 - Archived\\Routers",
  "Default Settings": "Sessions\\99 - System"
}
```

Numeric prefixes keep important folders first because PSM sorts names alphabetically.

Apply the map with Python's structured [`winreg`](https://docs.python.org/3/library/winreg.html) API:

```python
import json
from pathlib import Path
import urllib.parse
import winreg

BASE = r"Software\SimonTatham\PuTTY\Sessions"
folders = json.loads(Path("psm-folders.json").read_text())

with winreg.OpenKey(winreg.HKEY_CURRENT_USER, BASE) as root:
    raw_names = [
        winreg.EnumKey(root, index)
        for index in range(winreg.QueryInfoKey(root)[0])
    ]

sessions = {urllib.parse.unquote(raw): raw for raw in raw_names}

missing = set(folders) - set(sessions)
unmapped = set(sessions) - set(folders)
if missing or unmapped:
    raise SystemExit(
        f"Refusing partial update: missing={sorted(missing)}, "
        f"unmapped={sorted(unmapped)}"
    )

for display_name, folder in folders.items():
    path = BASE + "\\" + sessions[display_name]
    with winreg.OpenKey(
        winreg.HKEY_CURRENT_USER,
        path,
        0,
        winreg.KEY_SET_VALUE,
    ) as session:
        winreg.SetValueEx(session, "PsmPath", 0, winreg.REG_SZ, folder)
```

The coverage check is deliberate. A typo should produce no changes, not a partially reorganized tree.

## The `Default Settings` inheritance trap

PuTTY uses `Default Settings` as the template for new sessions. Some session generators copy every Registry value from that template.

Once `Default Settings` has a `PsmPath`, a naïve copy operation also copies the folder assignment. A newly generated session can then appear under `99 - System` even though nobody placed it there.

Treat presentation metadata as non-inheritable:

```python
value = winreg.EnumValue(defaults, index)
if value[0] != "PsmPath":
    copied_values.append(value)
```

The broader rule is useful outside PuTTY: when cloning configuration, distinguish behavioral settings from identity and presentation metadata.

## Configure the launcher path explicitly

PSM 0.50 defaults to an old 32-bit installation path:

```text
C:\Program Files (x86)\PuTTY\putty.exe
```

A current 64-bit PuTTY installation commonly lives here:

```text
C:\Program Files\PuTTY\putty.exe
```

If the path is wrong, PSM displays:

```text
PuTTY Failed to start.
Check the PuTTY location in System Tray -> Options.
```

PSM stores the per-user override in its .NET `user.config` below `%LOCALAPPDATA%`. Set `PuttyLocation` through the Options dialog or update that per-user setting while PSM is stopped.

## Tray behavior

PSM is designed as a tray application. Its window has no conventional minimize button; closing the window hides it while leaving the tray process running. It can also hide automatically after a session launch by enabling `MinimizeOnUse`.

That design makes the final workflow small:

1. Open PSM from the tray.
2. Select a session from the organized tree.
3. Launch it without changing its saved PuTTY name.
4. Let PSM return to the tray.
5. Maintain future folder changes in the JSON map and reapply them.

## Source

- [PuTTY Session Manager project site](https://puttysm.sourceforge.io/)
- [PuTTY Session Manager 0.50 download](https://sourceforge.net/projects/puttysm/files/latest/download)
- [PuTTY Session Manager source browser](https://sourceforge.net/p/puttysm/code/HEAD/tree/trunk/)
- [Python `winreg` documentation](https://docs.python.org/3/library/winreg.html)
