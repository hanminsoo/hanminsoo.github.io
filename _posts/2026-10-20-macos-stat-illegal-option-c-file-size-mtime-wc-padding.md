---
title: "macOS `stat: illegal option -- c`: file size and mtime with BSD `stat -f`, why `wc -c` breaks a string compare, and `find -printf: unknown primary`"
description: >-
  `stat -c %s file` fails on macOS with "stat: illegal option -- c" because
  /usr/bin/stat is BSD stat. Use `stat -f %z` for the size, `-f %m` for the
  mtime in epoch seconds, and `-t FORMAT -f %Sm` for a formatted mtime. The
  common portable fallback `$(wc -c < file)` returns "       6" with leading
  spaces on macOS, so `[ "$size" = 6 ]` is false. `find -printf` doesn't exist
  either ("unknown primary or operator"); use `find ... -exec stat -f ... {} +`.
  Reproduced on macOS 26.
tags: [macos, shell, stat, bsd]
---

A script that reads a file's size or modification time stops on a Mac:

```console
$ mkdir demo && cd demo
$ export TZ=UTC
$ printf 'hello\n' > a.txt && touch -t 202610010930.15 a.txt
$ stat -c %s a.txt; echo "exit $?"
stat: illegal option -- c
usage: stat [-FLnq] [-f format | -l | -r | -s | -x] [-t timefmt] [file ...]
exit 1
```

`-c FORMAT` is GNU stat. macOS ships BSD stat, which takes its format with
`-f` and uses different letters. Below: the translations I checked, a
`wc -c` workaround that quietly breaks comparisons, and what to use instead
of `find -printf`. (`TZ=UTC` and a fixed `touch -t` make the time outputs
the same on every run.)

There's no `--version`; `what` shows the build:

```console
$ stat --version; echo "exit $?"
stat: illegal option -- -
usage: stat [-FLnq] [-f format | -l | -r | -s | -x] [-t timefmt] [file ...]
exit 1
$ what /usr/bin/stat /usr/bin/wc /usr/bin/find
/usr/bin/stat:
	PROGRAM:stat  PROJECT:file_cmds-479
	PROGRAM:stat  PROJECT:file_cmds-479
/usr/bin/wc:
	PROGRAM:wc  PROJECT:text_cmds-199
	PROGRAM:wc  PROJECT:text_cmds-199
/usr/bin/find:
	PROGRAM:find  PROJECT:shell_cmds-329
	PROGRAM:find  PROJECT:shell_cmds-329
```

## BSD `stat -f`: size, mtime, name

```console
$ stat -f %z a.txt
6
$ stat -f %m a.txt
1790847015
$ stat -f %Sm a.txt
Oct  1 09:30:15 2026
$ stat -t '%F %T' -f %Sm a.txt
2026-10-01 09:30:15
$ stat -f '%N %z' a.txt
a.txt 6
```

- `%z` is the size in bytes (GNU `%s`).
- `%m` is the mtime in epoch seconds (GNU `%Y`).
- `%Sm` is the mtime as a string. The `S` asks for a string, and `-t` sets
  its `strftime` format.
- `%N` is the file name (GNU `%n`).

The GNU equivalents above come from GNU's documentation. I didn't run GNU
stat here.

## `wc -c` as a "portable" size: the leading spaces

A common way around `stat` differences is `wc -c < file`. On macOS, `wc`
pads its number:

```console
$ size=$(wc -c < a.txt); echo "[$size]"
[       6]
$ [ "$size" = 6 ] && echo equal || echo "not equal"
not equal
$ [ "$size" -eq 6 ] && echo "numeric equal"
numeric equal
```

The numeric test (`-eq`) ignores the spaces, but a string compare, a
`case`, a JSON field, or a file name built from `$size` gets them. Strip
them, or let arithmetic expansion do it:

```console
$ size=$(wc -c < a.txt | tr -d ' '); echo "[$size]"
[6]
$ echo "[$(( $(wc -c < a.txt) ))]"
[6]
```

## `find -printf`: unknown primary

```console
$ find . -name '*.txt' -printf '%s %p\n'; echo "exit $?"
find: -printf: unknown primary or operator
exit 1
$ find . -name '*.txt' -exec stat -f '%z %N' {} +
6 ./a.txt
```

`-exec ... {} +` passes many files to one `stat` call, so this doesn't start
one process per file.

## Scripts that run on both

The usual pattern is to try the GNU form, then fall back to BSD:

```console
$ stat -c %s a.txt 2>/dev/null || stat -f %z a.txt
6
```

On this Mac, `-c` fails and the BSD form answers. If the file doesn't exist,
both forms fail, and the error you see comes from the second command:

```console
$ stat -c %s nosuch 2>/dev/null || stat -f %z nosuch; echo "exit $?"
stat: nosuch: stat: No such file or directory
exit 1
```

One caution I can only take from the documentation, since I had no Linux
machine to test on: don't swap the order. On GNU stat, `-f` means "file
system status", so `stat -f %z` doesn't fail there. It prints something
else, and the fallback never runs.

If all you need is the size, `$(( $(wc -c < file) ))` avoids the padding
(checked on macOS; I didn't test it on Linux).

## Checklist

- `stat: illegal option -- c` on macOS: `/usr/bin/stat` is BSD stat, so use
  `-f FORMAT`.
- Size `stat -f %z`, mtime epoch `stat -f %m`, formatted mtime
  `stat -t '%F %T' -f %Sm`, name `%N`.
- `wc -c < file` on macOS returns leading spaces (`[       6]`). String
  compares fail. Use `$(( ... ))` or `tr -d ' '`.
- `find -printf` doesn't exist. Use `find ... -exec stat -f '...' {} +`.
- Fallback order: `stat -c ... 2>/dev/null || stat -f ...`, GNU first
  (GNU `-f` means something else; that part wasn't tested here).

*Tested on macOS 26.5.2 (Apple silicon) with `/usr/bin/stat`
(`PROGRAM:stat  PROJECT:file_cmds-479`), `/usr/bin/wc` (`text_cmds-199`),
`/usr/bin/find` (`shell_cmds-329`) in zsh 5.9 (arm64-apple-darwin25.0),
`TZ=UTC`. GNU coreutils/findutils and Linux were not tested.*
