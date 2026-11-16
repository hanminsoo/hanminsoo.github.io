---
title: "macOS `ps: etimes: keyword not found` — process age in seconds, and why `2>/dev/null` doesn't hide it"
description: >-
  `ps -o etimes= -p PID` (process age in seconds, from Linux procps) fails on a
  Mac with "ps: etimes: keyword not found" and exit 1. The error goes to
  stderr, but the list of valid keywords goes to stdout, so
  `age=$(ps -o etimes= -p "$pid" 2>/dev/null)` fills the variable with that
  list. `[ "$age" -gt 3600 ]` then fails with "integer expression expected",
  the `if` takes the else branch and the script exits 0. Use `etime`
  ([[dd-]hh:]mm:ss) and convert it with awk, or `lstart` with LC_ALL=C and
  `date -j -f`.
tags: [macos, ps, shell, process]
---

A check that restarts a worker once it has been running for more than an hour,
written for Linux, run on a Mac:

```console
$ ps -o etimes= -p 1; echo "exit $?"
ps: etimes: keyword not found
ps: no valid keywords; valid keywords:
%cpu %mem acflag acflg args blocked comm command cpu cputime etime f flags gid
group inblk inblock jobc ktrace ktracep lim login logname lstart majflt minflt
msgrcv msgsnd ni nice nivcsw nsignals nsigs nswap nvcsw nwchan oublk oublock
p_ru paddr pagein pcpu pending pgid pid pmem ppid pri prsna pstime putime re
rgid rgroup rss ruid ruser sess sig sigmask sl start stat state stime svgid
svuid tdev time tpgid tsess tsiz tt tty ucomm uid upr user usrpri utime vsize
vsz wchan wq wqb wql wqr xstat
exit 1
```

`etimes` ("elapsed time in seconds") is a Linux procps keyword. The `ps` that
ships with macOS (`/bin/ps`, adv_cmds-237 here) doesn't have it. That much is
an ordinary error. The trouble is what it does inside a script.

## The error is on stderr, the keyword list is on stdout

```console
$ ps -o etimes= -p 1 >/dev/null
ps: etimes: keyword not found
ps: no valid keywords; valid keywords:
$ ps -o etimes= -p 1 2>/dev/null | head -3
%cpu %mem acflag acflg args blocked comm command cpu cputime etime f flags gid
group inblk inblock jobc ktrace ktracep lim login logname lstart majflt minflt
msgrcv msgsnd ni nice nivcsw nsignals nsigs nswap nvcsw nwchan oublk oublock
```

Only the first two lines go to stderr. The seven lines of keywords go to
stdout, so `2>/dev/null` hides the part that says what went wrong and keeps the
part that ends up in your variable.

With other columns in the list, `ps` prints those first and then fails:

```console
$ ps -o pid,etimes -p 1 2>/dev/null; echo "exit $?"
  PID
    1
exit 1
```

## What it does to a script

```console
$ cat old-check.sh
#!/bin/sh
# restart the worker if it has been running for more than an hour
pid=$1
age=$(ps -o etimes= -p "$pid" 2>/dev/null)
if [ "$age" -gt 3600 ]; then
  echo "worker $pid is $age s old, restarting"
else
  echo "worker $pid is fine"
fi
$ ./old-check.sh 1; echo "exit $?"
./old-check.sh: line 5: [: %cpu %mem acflag acflg args blocked comm command cpu cputime etime f flags gid
group inblk inblock jobc ktrace ktracep lim login logname lstart majflt minflt
msgrcv msgsnd ni nice nivcsw nsignals nsigs nswap nvcsw nwchan oublk oublock
p_ru paddr pagein pcpu pending pgid pid pmem ppid pri prsna pstime putime re
rgid rgroup rss ruid ruser sess sig sigmask sl start stat state stime svgid
svuid tdev time tpgid tsess tsiz tt tty ucomm uid upr user usrpri utime vsize
vsz wchan wq wqb wql wqr xstat: integer expression expected
worker 1 is fine
exit 0
```

PID 1 is `launchd`, running for 35 days on this Mac. The check says "fine".
`[` can't compare the keyword list with 3600, so it fails, the `if` goes to the
`else` branch, and the script exits 0. Run from launchd or cron with the output
thrown away, this check never restarts anything, and nothing tells you.

## What macOS `ps` has instead: `etime` and `lstart`

```console
$ ps -o pid,etime,lstart,comm -p 1,$$
  PID     ELAPSED STARTED                      COMM
    1 35-20:05:15 Sun Oct 11 13:57:09 2026     /sbin/launchd
85518       00:00 Mon Nov 16 10:02:24 2026     zsh
```

- `etime` is the elapsed time as text. The man page only says "elapsed running
  time"; what I saw on this Mac is `mm:ss` (`00:00`), `hh:mm:ss` (`08:13:34`)
  and `dd-hh:mm:ss` (`35-20:05:15`).
- `lstart` is the start time, in strftime's `%c` format (man ps), followed by
  trailing spaces.

Both can be turned into seconds.

## Fix 1: convert `etime` with awk

```sh
# etimes PID -> seconds since the process started (macOS ps has no etimes)
etimes() {
  local e
  e=$(ps -o etime= -p "$1") || return 1
  echo "$e" | awk '{
    n = split($1, t, /[-:]/)
    s = t[n] + 60 * t[n-1]
    if (n >= 3) s += 3600 * t[n-2]
    if (n == 4) s += 86400 * t[1]
    print s
  }'
}
```

I saved it as `etimes.sh`. `split` on `-` or `:` gives 2, 3 or 4 parts,
counted from the right: seconds, minutes, hours, days.

The `|| return 1` matters: for a PID that doesn't exist, `ps` prints nothing
and exits 1, and without the check the function would return 0 with an empty
result — the same trap as before.

Why awk and not shell arithmetic: the fields have leading zeros, and bash
(including macOS `/bin/sh`, which is bash 3.2) reads `08` and `09` as bad
octal numbers:

```console
$ bash -c 'm=08; echo $((m * 60))'
bash: 08: value too great for base (error token is "08")
```

awk reads `08` as 8.

```console
$ . ./etimes.sh
$ etimes 1
3096315
$ etimes $$
0
$ etimes 99999; echo "exit $?"
exit 1
```

The check, using it:

```sh
#!/bin/sh
. "$(dirname "$0")/etimes.sh"
pid=$1
age=$(etimes "$pid") || { echo "no process $pid" >&2; exit 1; }
if [ "$age" -gt 3600 ]; then
  echo "worker $pid is $age s old, restarting"
else
  echo "worker $pid is $age s old, fine"
fi
```

```console
$ ./new-check.sh 1; echo "exit $?"
worker 1 is 3096315 s old, restarting
exit 0
$ ./new-check.sh $$; echo "exit $?"
worker 85518 is 0 s old, fine
exit 0
$ ./new-check.sh 99999; echo "exit $?"
no process 99999
exit 1
```

A missing process is now an error (exit 1) instead of "fine".

## Fix 2: `lstart` + `date -j -f` — set LC_ALL=C

`lstart` uses `%c`, which follows your locale. On a Mac set to Korean it looks
like this:

```console
$ LANG=ko_KR.UTF-8 ps -o lstart= -p 1
2026년 10월 11일 일요일 13시 57분 09초
```

and the BSD `date -j -f` parse that works in English fails:

```console
$ s=$(LANG=ko_KR.UTF-8 ps -o lstart= -p 1); date -j -f "%a %b %d %T %Y" "$s" +%s; echo "exit $?"
Failed conversion of ``2026년 10월 11일 일요일 13시 57분 09초'' using format ``%a %b %d %T %Y''
date: illegal time format
usage: date [-jnRu] [-I[date|hours|minutes|seconds|ns]] [-f input_fmt]
            [ -z output_zone ] [-r filename|seconds] [-v[+|-]val[y|m|w|d|H|M|S]]
            [[[[mm]dd]HH]MM[[cc]yy][.SS] | new_date] [+output_fmt]
exit 1
```

Even in English, the trailing spaces make `date` print a warning:

```console
$ s=$(ps -o lstart= -p 1); date -j -f "%a %b %d %T %Y" "$s" +%s
Warning: Ignoring 4 extraneous characters in date string (    )
1791694629
```

So force the C locale and strip the spaces:

```console
$ s=$(LC_ALL=C ps -o lstart= -p 1 | sed "s/ *$//"); echo "[$s]"
[Sun Oct 11 13:57:09 2026]
$ echo $(( $(date +%s) - $(date -j -f "%a %b %d %T %Y" "$s" +%s) ))
3096315
```

Same number as `etimes 1` above. I also compared the two on a process whose
`etime` was in the `hh:mm:ss` form (`08:13:34`): both gave 29614.

Fix 1 is shorter and doesn't depend on the locale; I'd use that one. `lstart`
is useful when you want the start time itself, not the age.

## Checklist

- macOS `ps` has no `etimes`: "ps: etimes: keyword not found", exit 1.
- The list of valid keywords goes to **stdout**. `$(ps ... 2>/dev/null)` gets
  that list, not an empty string.
- `[ "$age" -gt N ]` with a non-number fails, the `if` takes the `else`
  branch, and the script can still exit 0. Check `ps`'s exit status
  (`|| exit 1`) before comparing.
- Use `etime` (`mm:ss`, `hh:mm:ss`, `dd-hh:mm:ss`) and convert it with awk —
  not with `$(( ))`, which rejects `08` and `09` in bash.
- `lstart` follows the locale and has trailing spaces: `LC_ALL=C` and strip
  them before `date -j -f`.
- A missing PID gives empty output and exit 1 — make that an error, not "fine".

Related: [macOS `date: illegal option -- d`](/2026/10/macos-date-illegal-option-d-bsd-date-j-f-v/)
and [`while pgrep -f` wait loop never ends](/2026/10/pgrep-f-wait-loop-never-ends-wait-on-pid/).

*Tested on 2026-11-16 on macOS 26.5.2 (build 25F84, Apple silicon) with
`/bin/ps` (adv_cmds-237), `/bin/date`, `/usr/bin/awk`, zsh 5.9 and
bash 3.2.57 (`/bin/sh`), in `env -i` with
`PATH=/usr/bin:/bin:/usr/sbin:/sbin`. Not tested: Linux procps `ps`, Homebrew
GNU tools, older macOS versions.*
