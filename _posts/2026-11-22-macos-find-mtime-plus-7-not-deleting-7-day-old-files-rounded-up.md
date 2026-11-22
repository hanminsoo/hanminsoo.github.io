---
title: "macOS `find -mtime +7` does not delete 7-day-old files (keeps them until day 8) — man says \"rounded up\""
description: >-
  `find logs -name '*.log' -mtime +7 -delete` on a Mac left a log that was
  7 days 10 hours old, with exit 0. The macOS man page says the age is
  "rounded up to the next full 24-hour period", but on this Mac (default
  COMMAND_MODE) the whole days are counted and the rest is dropped: `-mtime +7` matches only files
  at least 8 days old, `-mtime 0` matches the last 24 hours. For "older than
  7 days" use `-mtime +6`, or BSD find's unit form `-mtime +7d` (exact
  seconds) or `-mmin +10080`. `-mtime 7d` without a sign practically never matches.
tags: [macos, find, shell, bsd]
---

A log cleanup that should keep one week of logs, run on a Mac:

```console
$ find logs -name '*.log' -mtime +7 -print -delete; echo "exit $?"
logs/eight-days.log
exit 0
$ ls logs
seven-days-10h.log
six-days.log
```

`seven-days-10h.log` is more than 7 days old, and it is still there. No
error, exit 0. It will be deleted tomorrow, when it is 8 days old. With one
log per day and the cleanup running at the same time every day, that means 8
old logs on disk instead of 7, and anything that must be gone "after 7 days"
stays up to a day longer.

`man find` on macOS makes this more confusing, because it suggests the
opposite:

```console
$ man find | col -b | grep -A4 '^     -mtime'
     -mtime n[smhdw]
	     If no units are specified, this primary evaluates to true if the
	     difference between the file last modification time and the time
	     find was started, rounded up to the next full 24-hour period, is
	     n 24-hour periods.
```

Rounded up, 7 days 10 hours would be 8 periods, and `+7` ("more than 7")
would match it. It doesn't.

## Setup

Three files in a scratch folder, made 6 days, 7 days 10 hours and 8 days
1 hour old with BSD `date -v` and `touch -t`:

```console
$ T=$(mktemp -d)
$ cd "$T"
$ mkdir logs
$ touch -t "$(date -v-6d +%Y%m%d%H%M.%S)" logs/six-days.log
$ touch -t "$(date -v-7d -v-10H +%Y%m%d%H%M.%S)" logs/seven-days-10h.log
$ touch -t "$(date -v-8d -v-1H +%Y%m%d%H%M.%S)" logs/eight-days.log
$ for f in logs/*.log; do echo "$f $(( ($(date +%s) - $(stat -f %m "$f")) / 3600 )) hours"; done
logs/eight-days.log 193 hours
logs/seven-days-10h.log 178 hours
logs/six-days.log 144 hours
$ find logs -name '*.log' -mtime +7
logs/eight-days.log
```

(`touch -d "8 days ago"` does not work on macOS — see
[macOS `touch -d "2 days ago"`](/2026/11/macos-touch-d-out-of-range-or-illegal-time-specification-days-ago/).)

178 hours is 7.4 days. `-mtime +7` skips it.

## What `-mtime n` actually counts here

To see the boundaries, a script that makes files a given number of hours
old and asks `find` about each of them:

```console
$ cat ages.sh
#!/bin/sh
# make files N hours old, then show which ones each -mtime test picks
mkdir -p ages && cd ages || exit 1
now=$(date +%s)
for h in 0 23 24 47 48 167 168 178 191 192; do
  touch -t "$(date -r $((now - h * 3600 - 30)) +%Y%m%d%H%M.%S)" "${h}h"
done
for t in 0 1 7 -1 -7 +0 +6 +7; do
  printf '%-3s ' "$t"
  find . -type f -mtime "$t" | sed 's|^\./||' | sort -n | tr '\n' ' '
  echo
done
$ sh ages.sh
0   0h 23h 
1   24h 47h 
7   168h 178h 191h 
-1  0h 23h 
-7  0h 23h 24h 47h 48h 167h 
+0  24h 47h 48h 167h 168h 178h 191h 192h 
+6  168h 178h 191h 192h 
+7  192h 
```

(Each file is 30 seconds older than its name says, so "24h" is 24 hours
30 seconds and no file sits exactly on a 24-hour boundary.)

The output fits one rule: **the age is divided by 24 hours and the remainder
is dropped**, then compared with `n`:

| test | matches (age) | "older than" in plain words |
|---|---|---|
| `-mtime 0` | less than 24 h | modified in the last day |
| `-mtime 1` | 24 h up to 48 h | |
| `-mtime 7` | 7 days up to 8 days | |
| `-mtime -7` | less than 7 days | newer than 7 days |
| `-mtime +0` | 24 h or more | older than 1 day |
| `-mtime +6` | 7 days or more | older than 7 days |
| `-mtime +7` | 8 days or more | older than **8** days |

If the age were rounded up as the man page says, the 0-hour file would be in
period 1 and `-mtime 0` would match nothing; here it matches the last 24
hours. The table is what `/usr/bin/find` did on this Mac with the default
`COMMAND_MODE` (unset; `COMMAND_MODE=unix2003` gave the same output).

So `+7` means "more than 7 *whole* days", and a file has 7 whole days only
from the start of its 8th day.

## Fix 1: `-mtime +6`

For "older than 7 days" in whole days, ask for more than 6:

```console
$ find logs -name '*.log' -mtime +6
logs/eight-days.log
logs/seven-days-10h.log
```

The cut is at 7 × 24 hours, at least to the minute. Checked with files one
minute on each side of it:

```console
$ mkdir edge
$ now=$(date +%s)
$ touch -t "$(date -r $((now - 604800 + 60)) +%Y%m%d%H%M.%S)" edge/7d-minus-1min
$ touch -t "$(date -r $((now - 604800 - 60)) +%Y%m%d%H%M.%S)" edge/7d-plus-1min
$ find edge -type f -mtime +6
edge/7d-plus-1min
$ find edge -type f -mtime -7
edge/7d-minus-1min
```

(604800 seconds = 7 × 24 × 3600.)

This assumes `COMMAND_MODE` is not set to `legacy`; see
[below](#command_modelegacy-does-round-up).

## Fix 2: a unit suffix (`-mtime +7d`), macOS/BSD only

macOS `find` also takes a unit after the number: `s`, `m`, `h`, `d`, `w`.
With a unit the man page says the comparison is against the exact age, with
no rounding, and that is what happened:

```console
$ find logs -name '*.log' -mtime +7d
logs/eight-days.log
logs/seven-days-10h.log
$ find edge -type f -mtime +7d
edge/7d-plus-1min
$ find edge -type f -mtime +168h
edge/7d-plus-1min
$ find edge -type f -mtime +1w
edge/7d-plus-1min
```

Two things to know about the unit form:

```console
$ find logs -name '*.log' -mtime 7d; echo "exit $?"
exit 0
$ find logs -mtime +7x; echo "exit $?"
find: -mtime: +7x: bad unit 'x'
exit 1
$ find logs -mtime +7.5; echo "exit $?"
find: -mtime: +7.5: bad unit '.'
exit 1
```

- Without `+` or `-`, `7d` means "exactly 604800 seconds old", which a real
  file practically never is. It matches nothing and exits 0.
- No fractions: `7.5` is read as `7` with the unit `.`. Units can be
  combined instead; `+7d12h` is 7.5 days. Files one minute on each side of
  7 days 12 hours:

```console
$ mkdir edge2
$ now=$(date +%s)
$ s=$((7 * 86400 + 12 * 3600))
$ touch -t "$(date -r $((now - s + 60)) +%Y%m%d%H%M.%S)" edge2/7d12h-minus-1min
$ touch -t "$(date -r $((now - s - 60)) +%Y%m%d%H%M.%S)" edge2/7d12h-plus-1min
$ find edge2 -type f -mtime +7d12h
edge2/7d12h-plus-1min
$ find edge2 -type f -mtime -7d12h
edge2/7d12h-minus-1min
```

The unit suffix is a BSD `find` feature. I have no GNU `find` or Linux
machine here and did not test either. Per the GNU findutils documentation,
GNU `-mtime` also ignores the fractional part of the day count, the
docs describe no unit suffix, and GNU `find` has `-mmin` too. Check on
your Linux machine before reusing the script there.

## Fix 3: minutes (`-mmin +10080`)

`-mmin` counts minutes instead of days, so the leftover is at most a minute:

```console
$ find logs -name '*.log' -mmin +10080
logs/eight-days.log
logs/seven-days-10h.log
$ find edge -type f -mmin +10080
edge/7d-plus-1min
```

(10080 = 7 × 24 × 60.)

## `COMMAND_MODE=legacy` does round up

Some macOS tools read the `COMMAND_MODE` environment variable, and with
`COMMAND_MODE=legacy` this `find` behaves like the man page. The same
`ages.sh` from above, run again (it touches its files afresh):

```console
$ COMMAND_MODE=legacy sh ages.sh
0   
1   0h 23h 
7   167h 
-1  0h 23h 
-7  0h 23h 24h 47h 48h 167h 
+0  0h 23h 24h 47h 48h 167h 168h 178h 191h 192h 
+6  167h 168h 178h 191h 192h 
+7  168h 178h 191h 192h 
```

Now `-mtime 0` matches nothing, `+7` matches anything over 7 days, and `+6`
already matches the 167-hour file. (The `-1` and `-7` rows came out the same
in both modes; I did not dig into why.) So the `+6` fix above is for the
default mode only: under `COMMAND_MODE=legacy` it also deletes files younger
than 7 days (here: 167 hours, and 7 days minus 1 minute).

```console
$ COMMAND_MODE=legacy find edge -type f -mtime +6
edge/7d-plus-1min
edge/7d-minus-1min
$ COMMAND_MODE=legacy find logs -name '*.log' -mtime +7
logs/eight-days.log
logs/seven-days-10h.log
```

(`edge` and `logs` as made above, before the cleanup script ran.)
`COMMAND_MODE` was unset in my login shell, but if a script must not depend
on it, use the unit form (Fix 2) or `-mmin` (Fix 3), which gave the same
answers in both modes:

```console
$ COMMAND_MODE=legacy find edge -type f -mtime +7d
edge/7d-plus-1min
$ COMMAND_MODE=legacy find edge -type f -mmin +10080
edge/7d-plus-1min
```

## The cleanup script

Before: keeps the 7-days-10-hours log.

```console
$ cat clean-logs.sh
#!/bin/sh
# delete logs older than 7 days
dir=${1:?usage: clean-logs.sh DIR}
find "$dir" -name '*.log' -type f -mtime +7 -print -delete
$ sh clean-logs.sh logs; echo "exit $?"
logs/eight-days.log
exit 0
$ ls logs
seven-days-10h.log
six-days.log
```

After: the same line with `+6`, and the number of days in one place so the
"minus one" is not a surprise to the next reader.

```console
$ touch -t "$(date -v-8d -v-1H +%Y%m%d%H%M.%S)" logs/eight-days.log
$ cat clean-logs.sh
#!/bin/sh
# delete logs older than KEEP_DAYS days (default 7)
dir=${1:?usage: clean-logs.sh DIR}
keep=${KEEP_DAYS:-7}
# -mtime +N means "N+1 whole days or more", so ask for keep-1
find "$dir" -name '*.log' -type f -mtime +$((keep - 1)) -print -delete
$ sh clean-logs.sh logs; echo "exit $?"
logs/eight-days.log
logs/seven-days-10h.log
exit 0
$ ls logs
six-days.log
```

`-print` before `-delete` puts every deleted name in the log. To try a new
age without deleting anything, run the same `find` without `-delete` first.

Clean up:

```console
$ cd /
$ rm -r "${T:?}"
```

## Checklist

- On this Mac, `find -mtime n` (no unit) divides the age by 24 hours and
  drops the remainder, despite "rounded up" in `man find`. `-mtime +7`
  matches files 8 days old or more. With `COMMAND_MODE=legacy` it rounds up
  instead, and `+6` then also matches files younger than 7 days.
- "Older than 7 days": `-mtime +6`. "Newer than 7 days": `-mtime -7`.
  "Last 24 hours" (default mode): `-mtime 0` (or `-mtime -1`).
- macOS/BSD `find` also accepts units: `-mtime +7d`, `+168h`, `+1w`, `+7d12h`
  compare exact ages (checked to the minute). Without a sign (`-mtime 7d`) nothing matches, exit 0.
- `-mmin +10080` is 7 days to the minute.
- A wrong age is silent: `find` exits 0 whether it deletes one file or none.
  Keep `-print` next to `-delete`, and test the boundary with files made by
  `touch -t "$(date -r $((now - seconds)) +%Y%m%d%H%M.%S)"`.

Related: [macOS `touch -d "2 days ago"`](/2026/11/macos-touch-d-out-of-range-or-illegal-time-specification-days-ago/)
and [macOS `ls --time-style`](/2026/11/macos-ls-time-style-unrecognized-option-long-iso-full-time-ls-d/)
(which uses `find -newermt` for a cutoff date).

*Tested on 2026-11-22 on macOS 26.5.2 (Apple silicon, APFS) with
`/usr/bin/find` and `/bin/date` (shell_cmds-329), `/usr/bin/touch` and
`/usr/bin/stat` (file_cmds-479), zsh 5.9 and `/bin/sh` (bash 3.2.57), in
`env -i` with `PATH=/usr/bin:/bin:/usr/sbin:/sbin` (`COMMAND_MODE` unset,
plus the `COMMAND_MODE=legacy` runs shown above).
Not tested: GNU findutils `find`, Linux, older macOS versions, files with a
future modification time, days that cross a daylight-saving change.*
