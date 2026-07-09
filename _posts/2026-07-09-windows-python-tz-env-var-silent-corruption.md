---
layout: post
title: "A TZ environment variable silently corrupts every local timestamp Python computes on Windows"
date: 2026-07-09
categories: windows python tools
---

`ping` to `1.1` (shorthand for `1.0.0.1`) was timing out, then failing outright with `General failure`, then working fine two minutes later after a Wi-Fi radio toggle. Chasing the root cause through Windows event logs turned up something worse than the original problem: a script correlating those logs was reporting timestamps that were **4.5 hours wrong**, silently, with no exception and no warning.

```
>>> import os
>>> os.environ['TZ'] = 'Asia/Kolkata'
>>> import time
>>> time.localtime()
time.struct_time(tm_year=2026, tm_mon=7, tm_mday=9, tm_hour=2, tm_min=28, ..., tm_isdst=1)
>>> time.gmtime()
time.struct_time(tm_year=2026, tm_mon=7, tm_mday=9, tm_hour=1, tm_min=28, ..., tm_isdst=0)
```

UTC (`gmtime`) is `01:28`. India Standard Time is UTC+5:30, so local time should read `06:58`. Python says `02:28` — UTC plus one hour, not five and a half. And `tm_isdst=1` is flatly wrong: India has never observed daylight saving time.

## Root cause: Windows' C runtime doesn't speak IANA

`time.localtime()` and bare `datetime.now()` go through the C runtime's `tzset()`. On Linux, `tzset()` reads `/usr/share/zoneinfo` and understands IANA names like `Asia/Kolkata` natively. On Windows, MSVCRT's `_tzset()` only understands a POSIX-style format: `tzn[+|-]hh[:mm[:ss]][dzn]` — e.g. `IST-5:30` for India, `MSK-3` for Moscow.

Handed `Asia/Kolkata`, an IANA name it can't parse, `_tzset()` does not raise an error. [Per Microsoft's own docs](https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/tzset?view=msvc-170) and confirmed by a CPython triager on the tracker, it silently falls back to a default: **UTC+0, with U.S. daylight-saving rules applied**. In July, U.S. DST is in effect, so that default resolves to UTC+1 — which is exactly the `+01:00` this box reported, and exactly why `tm_isdst` was set even though the real zone has no DST at all.

This is a known, closed-as-"third party" issue on the Python tracker — [bpo-44352](https://bugs.python.org/issue44352) — and it recurs periodically as new users hit it: [cpython#124286](https://github.com/python/cpython/issues/124286) is the same bug filed again in 2024 against a fresh European timezone. From the tracker, a Python core triager (Eryk Sun) explaining the exact mechanism:

> "The CRT does not verify that the TZ value is valid. 'Europe/Moscow' is invalid, but it's blindly parsed anyway. There's no UTC offset, so it defaults to UTC (0 offset). It's parsed as supporting daylight saving time, according to U.S. rules... I recommend that you unset the TZ environment variable before running Windows Python."

## Why this is worse than a crash

A wrong-but-plausible answer is more dangerous than an error. This didn't throw. It didn't log a warning. It returned a `datetime` object that type-checks fine, prints fine, and is silently off by an amount that depends on the calendar — the gap between UTC and your real zone, shifted further by whether U.S. DST happens to be active that day. Any code doing "is this timestamp within the last hour," "is it after 5pm," or correlating Python-side timestamps against timestamps from another source (Windows Event Log, a database, another process) gets corrupted math with no signal that anything is wrong.

The blast radius is bigger than one script, too: **any** Python process on a Windows machine with `TZ` set to an IANA name inherits this silently, regardless of what that process is doing.

## The fix

Don't rely on the CRT's `tzset()` path at all. Use `zoneinfo` (stdlib, PEP 615) instead of bare `datetime.now()`:

```python
from datetime import datetime
from zoneinfo import ZoneInfo

# Wrong on Windows when TZ=Asia/Kolkata is set — silently off by hours
now = datetime.now()

# Correct — reads real IANA tzdata directly, bypasses the broken CRT path
now = datetime.now(ZoneInfo("Asia/Kolkata"))
```

`zoneinfo` reads the system's own timezone database first, and falls back to the [`tzdata`](https://pypi.org/project/tzdata/) PyPI package (a pure-data port of the IANA database, officially recommended by the Python core team for exactly this Windows gap) when the OS doesn't ship one — which Windows doesn't. Neither of these paths goes through `_tzset()`, so the corruption never applies.

For auto-detecting the local zone rather than hardcoding it, [`tzlocal`](https://pypi.org/project/tzlocal/) does the right thing on Windows by reading the registry (the same source PowerShell's `Get-Date` and `[System.TimeZoneInfo]::Local` use) instead of the environment:

```python
import tzlocal
now = datetime.now(tzlocal.get_localzone())
```

Microsoft's own CRT documentation agrees with the tracker's advice: on Windows, the officially recommended fix is simpler still — don't set `TZ` at all, and let the CRT fall through to the OS's registry-based timezone, which it handles correctly. `TZ` should be treated as a Linux/POSIX convention that Windows Python only partially and dangerously supports.

## Quick self-check

If you're on Windows and have `TZ` set to anything other than a POSIX offset string, check whether it's actually correct right now:

```powershell
python -c "import time; print(time.tzname, time.localtime())"
Get-Date  # ground truth, reads the registry correctly
```

If the hour reported by Python doesn't match `Get-Date`, every local-time computation in every Python process on that machine is wrong, and has probably been wrong for a while without anyone noticing. It's been asked before and never really answered: a [Stack Overflow question from 2020](https://stackoverflow.com/questions/62004265/python-3-time-tzset-alternative-for-windows) with 8 upvotes asking for the Windows equivalent of `time.tzset()` sat with zero answers for six years.

## Source

- [bpo-44352](https://bugs.python.org/issue44352) — original tracker report and the core triager's explanation of the mechanism
- [StackOverflow #62004265](https://stackoverflow.com/questions/62004265/python-3-time-tzset-alternative-for-windows) — the same question, asked in 2020, still unanswered
- [cpython#124286](https://github.com/python/cpython/issues/124286) — the same bug recurring against a different timezone
- [Microsoft docs: `_tzset`](https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/tzset?view=msvc-170) — the documented (but unenforced) TZ string format
- [PEP 615](https://peps.python.org/pep-0615/) — `zoneinfo`, the stdlib fix
- [`tzdata`](https://pypi.org/project/tzdata/), [`tzlocal`](https://pypi.org/project/tzlocal/) — the two PyPI packages that make this correct and portable
