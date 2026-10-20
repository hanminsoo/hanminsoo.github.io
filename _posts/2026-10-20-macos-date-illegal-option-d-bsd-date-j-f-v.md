---
title: "macOS `date: illegal option -- d`: BSD `date -j -f` and `-v` instead of `date -d`, and three surprises (current time filled in, Feb 30 accepted, month-end clamping)"
description: >-
  `date -d yesterday` and `date -d 2026-10-20 +%s` fail on macOS with
  "date: illegal option -- d", because /bin/date is BSD date. The replacements
  are `-v` for relative dates and `-j -f FORMAT` for parsing. But `-j -f
  '%Y-%m-%d'` fills in the current time of day, so the epoch you get is not
  midnight. `2026-02-30` is accepted and becomes March 2, with exit 0. And
  `-v-1m` from March 31 gives February 28. Reproduced with /bin/date
  (shell_cmds-329) on macOS 26.
tags: [macos, shell, date, bsd]
---

A script that works on Linux stops on a Mac at its first date calculation:

```console
$ date -d yesterday +%F; echo "exit $?"
date: illegal option -- d
usage: date [-jnRu] [-I[date|hours|minutes|seconds|ns]] [-f input_fmt]
            [ -z output_zone ] [-r filename|seconds] [-v[+|-]val[y|m|w|d|H|M|S]]
            [[[[mm]dd]HH]MM[[cc]yy][.SS] | new_date] [+output_fmt]
exit 1
```

`-d` ("display this date") belongs to GNU date. macOS ships BSD date, which
does the same jobs with different flags: `-v` to step a date forward or back,
and `-j -f` to parse a date string. Below are the translations, and three
places where BSD date silently returns something you might not expect. All
commands were run in zsh with `TZ=UTC`, so the epoch numbers can be checked
anywhere.

```console
$ export TZ=UTC
$ date --version; echo "exit $?"
date: illegal option -- -
usage: date [-jnRu] [-I[date|hours|minutes|seconds|ns]] [-f input_fmt]
            [ -z output_zone ] [-r filename|seconds] [-v[+|-]val[y|m|w|d|H|M|S]]
            [[[[mm]dd]HH]MM[[cc]yy][.SS] | new_date] [+output_fmt]
exit 1
$ what /bin/date
/bin/date:
	PROGRAM:date  PROJECT:shell_cmds-329
	PROGRAM:date  PROJECT:shell_cmds-329
```

(There's no `--version`, so `what` is the way to see which build it is.)

## Relative dates: `-v`

`-v` adjusts one field: `-v-1d` is one day back, `-v+2w` two weeks ahead.
You can repeat it.

```console
$ date +%F; date -v-1d +%F; date -v+2w +%F
```

The output depends on the day you run it. Mine is shown in the
"Results" block at the end.

## Parsing a given date: `-j -f`

`-f FORMAT STRING` parses STRING. `-j` means "don't try to set the clock".
Without `-j`, date treats the input as a new time to set the system clock to.

```console
$ date -j -f '%Y-%m-%d' 2026-10-20 +%F; echo "exit $?"
2026-10-20
exit 0
```

### Surprise 1: the time of day is today's, not midnight

Convert that same date to epoch seconds and print the time, too:

```console
$ date -j -f '%Y-%m-%d' 2026-10-20 '+%F %T %s'
```

The time printed is the **time you ran the command**, not 00:00:00, and the
epoch changes on every run. Two runs about two seconds apart:

```console
$ TZ=UTC date -j -f '%Y-%m-%d' 2026-10-20 '+%H:%M:%S %s'
22:00:35 1792533635
$ TZ=UTC date -j -f '%Y-%m-%d' 2026-10-20 '+%H:%M:%S %s'
22:00:37 1792533637
```

 Fields that aren't in the format
are taken from the current time. If you compare dates by their epoch, or
compute "days between" by dividing by 86400, this shifts your numbers.

Put the time in the format and in the string:

```console
$ date -j -f '%Y-%m-%d %H:%M:%S' '2026-10-20 00:00:00' '+%F %T %s'
2026-10-20 00:00:00 1792454400
```

`1792454400` is 2026-10-20 00:00:00 UTC. It's the same on every run.

### Surprise 2: an impossible date is accepted

```console
$ date -j -f '%Y-%m-%d' 2026-02-30 +%F; echo "exit $?"
2026-03-02
exit 0
$ date -j -f '%Y-%m-%d' 2026-13-01 +%F; echo "exit $?"
Failed conversion of ``2026-13-01'' using format ``%Y-%m-%d''
date: illegal time format
usage: date [-jnRu] [-I[date|hours|minutes|seconds|ns]] [-f input_fmt]
            [ -z output_zone ] [-r filename|seconds] [-v[+|-]val[y|m|w|d|H|M|S]]
            [[[[mm]dd]HH]MM[[cc]yy][.SS] | new_date] [+output_fmt]
exit 1
```

Month 13 is rejected, but February 30 rolls over to March 2 with exit 0. If
you use `date -j -f` to check that a date is valid, convert it back and
compare it with the input:

```console
$ d=2026-02-30; [ "$(date -j -f '%Y-%m-%d' "$d" +%F)" = "$d" ] && echo valid || echo invalid
invalid
$ d=2026-02-28; [ "$(date -j -f '%Y-%m-%d' "$d" +%F)" = "$d" ] && echo valid || echo invalid
valid
```

### Surprise 3: month steps clamp to the end of the month

```console
$ date -j -v-1m -f '%Y-%m-%d' 2026-03-31 +%F
2026-02-28
$ date -j -v+1m -f '%Y-%m-%d' 2026-01-31 +%F
2026-02-28
```

One month before March 31 is February 28, and one month after January 31
is also February 28. Stepping back from 03-31 and then forward again
doesn't return to 03-31. I didn't compare this with GNU date, because it
isn't installed here.

## Translation table

I ran every command in the BSD column on this Mac. The GNU column is the
usual Linux form, written from GNU's documentation and not run here.

| GNU (Linux) | BSD (macOS) |
|---|---|
| `date -d yesterday +%F` | `date -v-1d +%F` |
| `date -d '+2 weeks' +%F` | `date -v+2w +%F` |
| `date -d 2026-10-20 +%s` | `date -j -f '%Y-%m-%d %H:%M:%S' '2026-10-20 00:00:00' +%s` |
| `date -d @1792454400 +%F` | `date -r 1792454400 +%F` |

```console
$ date -r 1792454400 '+%F %T'
2026-10-20 00:00:00
```

## Results

The outputs that depend on when the commands run. These are from my run:

```console
$ date +%F; date -v-1d +%F; date -v+2w +%F
2026-10-20
2026-10-19
2026-11-03
$ date -j -f '%Y-%m-%d' 2026-10-20 '+%F %T %s'
2026-10-20 21:59:47 1792533587
```

## Checklist

- `date: illegal option -- d` on macOS: `/bin/date` is BSD date, and `-d` is GNU only.
- Relative dates: `-v-1d`, `-v+2w`, `-v-1m` (one field per `-v`, repeatable).
- Parsing: `date -j -f FORMAT STRING +OUT`. Without `-j`, date tries to set
  the clock.
- `-j -f '%Y-%m-%d'` fills in the **current** time of day. For a stable
  epoch, include `%H:%M:%S` and `00:00:00`.
- `2026-02-30` is accepted as `2026-03-02` (exit 0). To validate, round-trip
  the date and compare it with the input.
- Moving a month-end date by `-v±1m` clamps to the end of the shorter month: 03-31 minus a month is 02-28.
- Epoch to date: `date -r SECONDS`.

*Tested on macOS 26.5.2 (Apple silicon) with `/bin/date` (`what /bin/date`:
`PROGRAM:date  PROJECT:shell_cmds-329`) in zsh 5.9 (arm64-apple-darwin25.0),
`TZ=UTC`. GNU date and Linux were not tested.*
