---
title: "macOS `touch -d \"2 days ago\"`: \"out of range or illegal time specification\" — and the cleanup test that passes anyway"
description: >-
  `touch -d "2 days ago" file` (or `-d "2026-11-01 10:00"`, `-d 2026-11-01`)
  fails on a Mac with "touch: out of range or illegal time specification:
  YYYY-MM-DDThh:mm:SS[.frac][tz]" and exit 1, and the file is not created.
  macOS touch takes only one absolute format for -d, has no --date, and
  silently turns Feb 30 into Mar 2. A test that backdates a log to check
  `find -mtime +7 -delete` then prints PASS without testing anything. Fixes:
  `touch -t "$(date -v-8d +%Y%m%d%H%M.%S)"`, `-d` with `date -v`, `-A`, `-r`.
tags: [macos, touch, shell, bsd]
---

Making a file look old, on a Mac:

```console
$ touch -d "2 days ago" old.log; echo "exit $?"
touch: out of range or illegal time specification: YYYY-MM-DDThh:mm:SS[.frac][tz]
exit 1
$ ls old.log
ls: old.log: No such file or directory
```

GNU `touch -d` takes free-form dates like `"2 days ago"`, `yesterday` or
`"2026-11-01 10:00"` (that is what the GNU manual describes; I did not test it
on Linux). The `touch` that ships with macOS (`/usr/bin/touch`, file_cmds-479
here) has a `-d` too, but it accepts exactly one format, the one printed in the
error: `YYYY-MM-DDThh:mm:SS`, with optional fraction and `Z`. Anything else is
"out of range or illegal time specification", and **the file is not created**.

## The test that passes without testing anything

One common reason to backdate a file is to test a cleanup job. Here is a small
test for `find -mtime +7 -delete`, run on a Mac:

```console
$ mkdir logs
$ cat test-cleanup.sh
#!/bin/sh
# make an 8-day-old log and a fresh one, run the cleanup, check the result
touch -d "8 days ago" logs/old.log
touch logs/new.log
find logs -name "*.log" -mtime +7 -delete
[ ! -e logs/old.log ] && echo "PASS: old log removed"
[ -e logs/new.log ] && echo "PASS: new log kept"
$ sh test-cleanup.sh; echo "exit $?"
touch: out of range or illegal time specification: YYYY-MM-DDThh:mm:SS[.frac][tz]
PASS: old log removed
PASS: new log kept
exit 0
```

`old.log` was never created, so "old log removed" is true, and the script
exits 0. The `touch` error is the only sign, and in CI logs or a script that
sends stderr to `/dev/null` it is easy to miss.

## What `-d` accepts

```console
$ touch -d 2026-11-01T10:00:00 a; echo "exit $?"
exit 0
$ touch -d "2026-11-01 10:00:00" b; echo "exit $?"
exit 0
$ touch -d 2026-11-01T10:00:00.5 c; echo "exit $?"
exit 0
$ touch -d 2026-11-01 d; echo "exit $?"
touch: out of range or illegal time specification: YYYY-MM-DDThh:mm:SS[.frac][tz]
exit 1
$ touch -d 2026-11-01T10:00 e; echo "exit $?"
touch: out of range or illegal time specification: YYYY-MM-DDThh:mm:SS[.frac][tz]
exit 1
$ touch -d 2026-11-01T10:00:00+09:00 f; echo "exit $?"
touch: out of range or illegal time specification: YYYY-MM-DDThh:mm:SS[.frac][tz]
exit 1
$ touch -d @1700000000 g; echo "exit $?"
touch: out of range or illegal time specification: YYYY-MM-DDThh:mm:SS[.frac][tz]
exit 1
$ touch --date=2026-11-01T10:00:00 h; echo "exit $?"
touch: illegal option -- -
usage: touch [-A [-][[hh]mm]SS] [-achm] [-r file] [-t [[CC]YY]MMDDhhmm[.SS]]
       [-d YYYY-MM-DDThh:mm:SS[.frac][tz]] file ...
exit 1
```

So, on this Mac:

- Date **and** time **with seconds** are required. A date alone or `hh:mm`
  without seconds fails.
- A space works in place of the `T` (as above); a lowercase `t` does not.
- The only time zone accepted is `Z`. Numeric offsets like `+09:00` fail.
- `@epoch`, relative dates and `--date=` do not exist.

Without `Z` the time is local; with `Z` it is UTC:

```console
$ TZ=America/New_York touch -d 2026-10-15T10:00:00 local.txt
$ TZ=America/New_York touch -d 2026-10-15T10:00:00Z utc.txt
$ TZ=UTC stat -f '%N %Sm' -t '%F %T %Z' local.txt utc.txt
local.txt 2026-10-15 14:00:00 UTC
utc.txt 2026-10-15 10:00:00 UTC
```

And "out of range" does not mean the date is checked. February 30 is accepted
and quietly becomes March 2:

```console
$ touch -d 2026-02-30T10:00:00 feb.txt; echo "exit $?"
exit 0
$ stat -f '%Sm' -t '%F %T' feb.txt
2026-03-02 10:00:00
```

## Fix: compute the date with `date -v`, pass it to `-t`

macOS `date` does the relative arithmetic (`-v-8d` = 8 days earlier), and
`touch -t` takes `[[CC]YY]MMDDhhmm[.SS]`:

```console
$ touch -t "$(date -v-8d +%Y%m%d%H%M.%S)" old.log; echo "exit $?"
exit 0
```

The same date in the `-d` format works too:

```console
$ touch -d "$(date -v-8d +%Y-%m-%dT%H:%M:%S)" old2.log; echo "exit $?"
exit 0
```

`-t` is the one I would use in scripts: GNU `touch` documents the same `-t`
format (I did not test it). The `date -v` part is macOS/BSD only; GNU `date`
uses `-d "8 days ago"` instead.

`-mtime +7` means "more than 7 whole days old", so with the 8-day-old file the
find matches:

```console
$ stat -f '%Sm' -t '%F' old.log; date -v-8d +%F
2026-11-10
2026-11-10
$ find . -name 'old*.log' -mtime +7 | sort
./old.log
./old2.log
```

(The dates are from the day I ran it; yours will be today minus 8 days.)

## Other ways: `-A` and `-r`

`-A` shifts the times of an **existing** file by `[-][[hh]mm]SS`:

```console
$ touch a.log; touch -A -480000 a.log; echo "exit $?"
exit 0
$ touch b.log; touch -A -1920000 b.log; echo "exit $?"
touch: Invalid offset spec, must be [-][[HH]MM]SS
exit 1
```

`-480000` is 48 hours back. The hours field is two digits, so 99 hours
(`-995959`) is the furthest it goes in one call; 8 days (192 hours) does not
fit.

`-r` copies the times from another file:

```console
$ touch -r old.log same-age.log; stat -f '%N %Sm' -t '%F' old.log same-age.log
old.log 2026-11-10
same-age.log 2026-11-10
```

## The test, fixed

Fail loudly when the setup fails, and check the old file really is old before
running the cleanup:

```console
$ cat test-cleanup.sh
#!/bin/sh
set -e
# make an 8-day-old log and a fresh one, run the cleanup, check the result
touch -t "$(date -v-8d +%Y%m%d%H%M.%S)" logs/old.log
touch logs/new.log
[ -n "$(find logs/old.log -mtime +7)" ] || { echo "SETUP FAILED: old.log is not old" >&2; exit 1; }
find logs -name "*.log" -mtime +7 -delete
[ ! -e logs/old.log ] && echo "PASS: old log removed"
[ -e logs/new.log ] && echo "PASS: new log kept"
$ sh test-cleanup.sh; echo "exit $?"
PASS: old log removed
PASS: new log kept
exit 0
```

With the old `touch -d "8 days ago"` line put back, the same script now stops
at the setup:

```console
$ sed -i '' 's/^touch -t .*old.log$/touch -d "8 days ago" logs\/old.log/' test-cleanup.sh
$ sh test-cleanup.sh; echo "exit $?"
touch: out of range or illegal time specification: YYYY-MM-DDThh:mm:SS[.frac][tz]
exit 1
```

## Checklist

- macOS `touch -d` takes only `YYYY-MM-DDThh:mm:SS[.frac][Z]` (a space instead
  of `T` also works). `"2 days ago"`, `yesterday`, a date alone, `hh:mm`
  without seconds, `+09:00`, `@epoch` and `--date=` all fail with "out of range
  or illegal time specification" or "illegal option".
- On that error the file is **not created**. A test that then checks "the old
  file is gone" passes without testing anything.
- Without `Z` the time is local, with `Z` UTC.
- Invalid days are not rejected: `2026-02-30T10:00:00` becomes March 2.
- Backdate with `touch -t "$(date -v-8d +%Y%m%d%H%M.%S)" file`.
- `touch -A -HHMMSS` shifts an existing file, up to 99 hours per call;
  `touch -r ref file` copies another file's times.
- In tests, `set -e` and check that the setup did what you think before
  checking the result.

Related: [macOS `date: illegal option -- d`](/2026/10/macos-date-illegal-option-d-bsd-date-j-f-v/)
and [macOS `stat: illegal option -- c`](/2026/10/macos-stat-illegal-option-c-file-size-mtime-wc-padding/).

*Tested on 2026-11-18 on macOS 26.5.2 (Apple silicon) with `/usr/bin/touch`
(file_cmds-479), `/bin/date`, `/usr/bin/stat`, `/usr/bin/find`, zsh 5.9 and
`/bin/sh` (bash 3.2.57), in `env -i` with `PATH=/usr/bin:/bin:/usr/sbin:/sbin`.
Not tested: GNU coreutils `touch`/`date`, Linux, older macOS versions.*
