---
title: "macOS `ls --time-style=long-iso`: \"unrecognized option\" (exit 1) — and the stale-file check that finds nothing"
description: >-
  `ls -l --time-style=long-iso` (or `--time-style=+%F`, `--full-time`) fails
  on a Mac with "ls: unrecognized option" and exit 1. In a script,
  `ls -l --time-style=+%F dir | awk '$6 < cutoff'` prints nothing, the
  pipeline exits 0, and a "stale backups" check reports "no stale backups"
  while two files are more than a week old. macOS ls uses `-D format` (strftime) and
  `-T` instead; `ls -lD dir` silently takes the folder name as the format.
  Parsing ls still breaks on spaces in names; use `find -newermt` + `stat -f`.
tags: [macos, ls, shell, bsd]
---

Asking `ls` for ISO dates, on a Mac:

```console
$ ls -l --time-style=long-iso backups; echo "exit $?"
ls: unrecognized option `--time-style=long-iso'
usage: ls [-@ABCFGHILOPRSTUWXabcdefghiklmnopqrstuvwxy1%,] [--color=when] [-D format] [file ...]
exit 1
$ ls -l --time-style=+%F backups; echo "exit $?"
ls: unrecognized option `--time-style=+%F'
usage: ls [-@ABCFGHILOPRSTUWXabcdefghiklmnopqrstuvwxy1%,] [--color=when] [-D format] [file ...]
exit 1
$ ls -l --full-time backups; echo "exit $?"
ls: unrecognized option `--full-time'
usage: ls [-@ABCFGHILOPRSTUWXabcdefghiklmnopqrstuvwxy1%,] [--color=when] [-D format] [file ...]
exit 1
```

`--time-style` and `--full-time` are GNU `ls` options. The `ls` that ships
with macOS (`/bin/ls`) does not have them; the only long option in its usage
line is `--color`. The replacement is in that same usage line: `-D format`.
On a terminal the error is obvious. In a script it can turn into a quiet
"all good".

## Setup

Three files in a scratch folder: one from today, one from 11 days ago, one
from last year with a space in its name.

```console
$ T=$(mktemp -d)
$ cd "$T"
$ mkdir backups
$ touch backups/new.tar
$ touch -t 202611101200 backups/old.tar
$ touch -t 202511010930 'backups/old notes.tar'
$ ls -l backups
total 0
-rw-r--r--  1 me  staff  0 Nov 21 10:03 new.tar
-rw-r--r--  1 me  staff  0 Nov  1  2025 old notes.tar
-rw-r--r--  1 me  staff  0 Nov 10 12:00 old.tar
```

(I ran this on 2026-11-21, so the 7-day cutoff below is 2026-11-14. On
another day the times, the cutoff and the `new.tar` line will differ. The user
name in the `ls -l` lines is replaced with `me`.)

The default format is why people reach for `--time-style` in the first
place: files from the last six months show a time (`10:03`), older ones show
the year in the same column (`2025`), and the month name follows the locale:

```console
$ LC_ALL=ko_KR.UTF-8 ls -l backups
total 0
-rw-r--r--  1 me  staff  0 11월 21 10:03 new.tar
-rw-r--r--  1 me  staff  0 11월  1  2025 old notes.tar
-rw-r--r--  1 me  staff  0 11월 10 12:00 old.tar
```

## The stale-backup check that says "no stale backups"

A small check: list backups that have not changed in the last 7 days.

```console
$ cat stale.sh
#!/bin/sh
# list backups not changed in the last 7 days
dir=$1
cutoff=$(date -v-7d +%F)
old=$(ls -l --time-style=+%F "$dir" | awk -v c="$cutoff" 'NR > 1 && $6 < c { print $7 }')
if [ -z "$old" ]; then
  echo "no stale backups"
else
  echo "stale:"
  echo "$old"
fi
$ sh stale.sh backups; echo "exit $?"
ls: unrecognized option `--time-style=+%F'
usage: ls [-@ABCFGHILOPRSTUWXabcdefghiklmnopqrstuvwxy1%,] [--color=when] [-D format] [file ...]
no stale backups
exit 0
```

`old.tar` (11 days) and `old notes.tar` (last year) are both stale. The script
says there are none and exits 0. Why:

- `ls` prints its error and usage line to stderr and nothing to stdout, so
  `awk` reads no lines and prints nothing.
- The pipeline's status is `awk`'s, which is 0:

```console
$ ls -l --time-style=+%F backups | awk 'NR > 1 { print $6, $7 }'; echo "exit $?"
ls: unrecognized option `--time-style=+%F'
usage: ls [-@ABCFGHILOPRSTUWXabcdefghiklmnopqrstuvwxy1%,] [--color=when] [-D format] [file ...]
exit 0
```

- An empty `$old` is exactly what "nothing is stale" looks like, so the
  `if` takes the happy branch.

If this runs from cron or launchd, the two stderr lines may go to a log
nobody reads (or nowhere at all), and the check reports success every day.

## macOS ls: `-D format` and `-T`

`-D` takes a `strftime(3)` format (see `man strftime`) and is used with `-l`:

```console
$ ls -lD %F backups
total 0
-rw-r--r--  1 me  staff  0 2026-11-21 new.tar
-rw-r--r--  1 me  staff  0 2025-11-01 old notes.tar
-rw-r--r--  1 me  staff  0 2026-11-10 old.tar
$ ls -lD '%Y-%m-%d %H:%M' backups
total 0
-rw-r--r--  1 me  staff  0 2026-11-21 10:03 new.tar
-rw-r--r--  1 me  staff  0 2025-11-01 09:30 old notes.tar
-rw-r--r--  1 me  staff  0 2026-11-10 12:00 old.tar
$ ls -lD '%F %T' backups
total 0
-rw-r--r--  1 me  staff  0 2026-11-21 10:03:16 new.tar
-rw-r--r--  1 me  staff  0 2025-11-01 09:30:00 old notes.tar
-rw-r--r--  1 me  staff  0 2026-11-10 12:00:00 old.tar
$ ls -lT backups
total 0
-rw-r--r--  1 me  staff  0 Nov 21 10:03:16 2026 new.tar
-rw-r--r--  1 me  staff  0 Nov  1 09:30:00 2025 old notes.tar
-rw-r--r--  1 me  staff  0 Nov 10 12:00:00 2026 old.tar
$ LC_ALL=ko_KR.UTF-8 ls -lD %F backups
total 0
-rw-r--r--  1 me  staff  0 2026-11-21 new.tar
-rw-r--r--  1 me  staff  0 2025-11-01 old notes.tar
-rw-r--r--  1 me  staff  0 2026-11-10 old.tar
```

So for the GNU spellings:

| GNU `ls` | macOS `ls` |
|---|---|
| `--time-style=+FORMAT` | `-D FORMAT` |
| `--time-style=long-iso` | `-D '%Y-%m-%d %H:%M'` |
| `--full-time` | `-T` (month name, seconds, year) or `-D '%F %T'` |

`-D '%F %T'` has no fractional seconds or time zone. The man page says `-D`
overrides `-T`, and a different format can mean a different number of columns
(`'%F %T'` is two columns, `%F` one).

Two things that are easy to get wrong:

```console
$ ls -D %F backups
new.tar
old notes.tar
old.tar
$ ls -lD backups; echo "exit $?"
total 8
drwxr-xr-x  5 me  staff  160 backups backups
-rw-r--r--  1 me  staff  265 backups stale.sh
exit 0
```

- Without `-l`, `-D` does nothing (no error).
- `-D` needs its argument. In `ls -lD backups` the folder name **is** the
  format, so `ls` lists the current folder with the literal word `backups`
  as every date, and exits 0.

## `-D` alone does not fix the script

Swapping `--time-style=+%F` for `-D %F` in the same pipeline:

```console
$ ls -lD %F backups | awk -v c="$(date -v-7d +%F)" 'NR > 1 && $6 < c { print $7 }'
old
old.tar
```

The dates are right now, but `$7` is only the first word of the name:
`old notes.tar` comes out as `old`. Anything done with that list (`rm`, `mv`,
an alert) acts on the wrong name. The column layout also depends on the
format string, and the error case still exits 0. Better not to parse `ls` at
all.

## The fix: `find -newermt` and `stat -f`

macOS `find` can compare modification times with a date string, and `stat`
can print the date in any `strftime` format:

```console
$ date -v-7d +%F
2026-11-14
$ find backups -maxdepth 1 -type f ! -newermt "$(date -v-7d +%F)"
backups/old notes.tar
backups/old.tar
$ find backups -maxdepth 1 -type f ! -newermt "$(date -v-7d +%F)" -exec stat -f '%Sm  %N' -t %F {} +
2025-11-01  backups/old notes.tar
2026-11-10  backups/old.tar
```

- `-newermt 2026-11-14` is true for files modified after 2026-11-14 00:00
  local time; `!` turns it into "not modified since then" — almost the same
  cut as `$6 < cutoff` above. The one difference is a file modified at exactly
  00:00:00 on the cutoff day: `$6 < cutoff` keeps it, `! -newermt` counts it
  as stale:
  ```console
  $ touch -t 202611140000.00 a; touch -t 202611140000.01 b
  $ find . -maxdepth 1 -type f ! -newermt 2026-11-14
  ./a
  ```
- `-maxdepth 1` keeps it to the folder itself, like `ls`.
- `stat -f '%Sm'` is the modification time as a string, `-t %F` its format,
  `%N` the name with its spaces intact.

The fixed script:

```console
$ cat stale.sh
#!/bin/sh
# list backups not changed in the last 7 days
dir=${1:?usage: stale.sh DIR}
cutoff=$(date -v-7d +%F) || exit 2
old=$(find "$dir" -maxdepth 1 -type f ! -newermt "$cutoff" \
  -exec stat -f '%Sm  %N' -t %F {} +) || {
  echo "could not list $dir" >&2
  exit 2
}
if [ -z "$old" ]; then
  echo "no stale backups (nothing before $cutoff)"
else
  echo "stale (before $cutoff):"
  echo "$old"
fi
$ sh stale.sh backups; echo "exit $?"
stale (before 2026-11-14):
2025-11-01  backups/old notes.tar
2026-11-10  backups/old.tar
exit 0
$ sh stale.sh nosuch; echo "exit $?"
find: nosuch: No such file or directory
could not list nosuch
exit 2
$ mkdir fresh
$ touch fresh/today.tar
$ sh stale.sh fresh; echo "exit $?"
no stale backups (nothing before 2026-11-14)
exit 0
$ sh stale.sh; echo "exit $?"
stale.sh: line 3: 1: usage: stale.sh DIR
exit 1
```

There is no pipe in the command substitution any more, so `find`'s own exit
status reaches the `||`: a missing folder is an error (exit 2), not "no stale
backups". The "nothing before" line also prints the cutoff, so a wrong date
is visible in the log.

`date -v-7d` is BSD `date`; the GNU form `date -d '7 days ago'` does not work
here (see
[macOS `touch -d "2 days ago"`](/2026/11/macos-touch-d-out-of-range-or-illegal-time-specification-days-ago/)).

Clean up:

```console
$ cd /
$ rm -r "${T:?}"
```

## Checklist

- macOS `ls` has no `--time-style` and no `--full-time`: "unrecognized
  option", exit 1. The only long option in its usage line is `--color`.
- The error goes to stderr. `ls -l --time-style=... | awk ...` prints nothing
  and the pipeline exits 0, which a check can read as "nothing found".
- Use `ls -lD FORMAT` (strftime): `-D %F`, `-D '%Y-%m-%d %H:%M'` for
  long-iso, `-D '%F %T'` or `-lT` for full time. `-D` ignores the locale for
  numeric formats like `%F`; the default `ls -l` dates do not.
- `-D` does nothing without `-l`, and `ls -lD dir` uses `dir` as the format
  (exit 0).
- Do not parse `ls -l` in scripts: names with spaces split. For "files older
  than a date" use `find DIR -maxdepth 1 -type f ! -newermt YYYY-MM-DD`, and
  `stat -f '%Sm %N' -t FORMAT` to print dates.

Related: [macOS `du -b`: "invalid option -- b"](/2026/11/macos-du-b-invalid-option-size-in-bytes-apparent-size/)
and [macOS `stat: illegal option -- c`](/2026/10/macos-stat-illegal-option-c-file-size-mtime-wc-padding/).

*Tested on 2026-11-21 on macOS 26.5.2 (Apple silicon, APFS) with `/bin/ls`,
`/usr/bin/stat` and `/usr/bin/touch` (file_cmds-479), `/usr/bin/find`
and `/bin/date` (shell_cmds-329), `/usr/bin/awk`, zsh 5.9 and `/bin/sh`
(bash 3.2.57), in `env -i` with `PATH=/usr/bin:/bin:/usr/sbin:/sbin`.
Not tested: GNU coreutils `ls` (`gls`), Linux, older macOS versions.*
