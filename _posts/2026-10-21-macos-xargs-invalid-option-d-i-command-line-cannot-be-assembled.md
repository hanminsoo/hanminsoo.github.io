---
title: "macOS `xargs: invalid option -- d` / `-- i`: one item per line with `tr '\\n' '\\0' | xargs -0`, and `-I {}` failing with \"command line cannot be assembled, too long\""
description: >-
  On macOS, `xargs -d '\n'` and `xargs -i` fail with "xargs: invalid option"
  because /usr/bin/xargs is BSD xargs. Without `-d`, a line like "my file.txt"
  is split into two arguments and an apostrophe stops xargs with
  "unterminated quote". `tr '\n' '\0' | xargs -0` keeps each line whole, and
  `-I {}` replaces `-i`. In my tests, `-I` refused any argument of 255 bytes or
  more after replacement ("command line cannot be assembled, too long"),
  stopped at that line, and exited 1; `-S 4096` raised the limit. Reproduced
  on macOS 26.
tags: [macos, shell, xargs, bsd]
---

A script that hands a list of file names to `xargs`, one per line, stops on
a Mac:

```console
$ mkdir demo && cd demo
$ printf 'my file.txt\nother.txt\n' > list.txt
$ xargs -d '\n' -n1 echo < list.txt; echo "exit $?"
xargs: invalid option -- d
usage: xargs [-0opt] [-E eofstr] [-I replstr [-R replacements] [-S replsize]]
             [-J replstr] [-L number] [-n number [-x]] [-P maxprocs]
             [-s size] [utility [argument ...]]
exit 1
$ xargs -i echo "[{}]" < list.txt; echo "exit $?"
xargs: invalid option -- i
usage: xargs [-0opt] [-E eofstr] [-I replstr [-R replacements] [-S replsize]]
             [-J replstr] [-L number] [-n number [-x]] [-P maxprocs]
             [-s size] [utility [argument ...]]
exit 1
```

`-d` (delimiter) and `-i` (lowercase, "replace `{}`") are GNU xargs options
(from GNU's documentation; I didn't run GNU xargs here). macOS ships BSD
xargs, which has neither. The usage line above lists what it
does take. There's no `--version`; `what` shows the build:

```console
$ xargs --version; echo "exit $?"
xargs: unrecognized option `--version'
usage: xargs [-0opt] [-E eofstr] [-I replstr [-R replacements] [-S replsize]]
             [-J replstr] [-L number] [-n number [-x]] [-P maxprocs]
             [-s size] [utility [argument ...]]
exit 1
$ what /usr/bin/xargs
/usr/bin/xargs:
	PROGRAM:xargs  PROJECT:shell_cmds-329
	PROGRAM:xargs  PROJECT:shell_cmds-329
```

## Why not just drop `-d`

Without `-d`, xargs splits on blanks and reads quotes:

```console
$ xargs -n1 echo < list.txt
my
file.txt
other.txt
$ printf "it's.txt\n" | xargs echo; echo "exit $?"
xargs: unterminated quote
exit 1
```

`my file.txt` became two arguments, and an apostrophe in a name stopped
xargs before it ran anything.

## One item per line: `tr '\n' '\0' | xargs -0`

With `-0`, xargs splits only on NUL; in the tests below it also left quotes
and backslashes alone. Turning newlines into NULs first gives one argument
per line:

```console
$ tr '\n' '\0' < list.txt | xargs -0 -n1 echo
my file.txt
other.txt
$ printf "it's.txt\n" | tr '\n' '\0' | xargs -0 echo
it's.txt
```

A last line without a final newline still came through:

```console
$ printf 'a\nb' | tr '\n' '\0' | xargs -0 -n1 echo
a
b
```

If the list comes from `find`, `-print0` writes the NULs itself:

```console
$ mkdir names && touch "names/my file.txt" "names/it's.txt"
$ find names -type f -print0 | xargs -0 -n1 echo
names/it's.txt
names/my file.txt
```

## Instead of `-i`: `-I {}`

`-I` takes the placeholder as an argument and runs the command once per
line:

```console
$ xargs -I {} echo "[{}]" < list.txt
[my file.txt]
[other.txt]
```

On its own, `-I` still treated the input differently from the raw line in
these cases: it dropped leading blanks, read quotes, and ate a backslash.
With `tr` and `-0` in front, the line came through as it was:

```console
$ printf '  lead.txt\n' | xargs -I {} echo "[{}]"
[lead.txt]
$ printf '  lead.txt\n' | tr '\n' '\0' | xargs -0 -I {} echo "[{}]"
[  lead.txt]
$ printf "it's.txt\n" | xargs -I {} echo "[{}]"; echo "exit $?"
xargs: unterminated quote
exit 1
$ printf 'a\\b\n' | xargs -I {} echo "[{}]"
[ab]
$ printf 'a\\b\n' | tr '\n' '\0' | xargs -0 -I {} echo "[{}]"
[a\b]
```

## `-I` and "command line cannot be assembled, too long"

BSD `xargs -I` has a separate size limit for the replaced argument. The man
page:

```console
$ man xargs | col -b | grep -A2 '^ *-S replsize'
     -S replsize
	     Specify the amount of space (in bytes) that -I can use for
	     replacements.  The default for replsize is 255.
```

In my tests, an argument of 254 bytes after replacement worked and 255 did
not. Lines of 254 and 255 `x` characters (`wc -c` counts `echo`'s newline
too):

```console
$ printf '%254s\n' | tr ' ' x | xargs -I {} echo {} | wc -c
     255
$ printf '%255s\n' | tr ' ' x | xargs -I {} echo {} | wc -c
xargs: command line cannot be assembled, too long
       0
```

The text around `{}` counts. With `pre-` in front, 250 characters of input
worked and 251 failed:

```console
$ printf '%250s\n' | tr ' ' x | xargs -I {} echo "pre-{}" | wc -c
     255
$ printf '%251s\n' | tr ' ' x | xargs -I {} echo "pre-{}" | wc -c
xargs: command line cannot be assembled, too long
       0
```

When one long line sits in the middle of the input, xargs stopped there.
The lines after it didn't run, and the exit status was 1:

```console
$ { echo a; printf '%300s\n' | tr ' ' x; echo b; } > mixed.txt
$ xargs -I {} echo "[{}]" < mixed.txt; echo "exit $?"
[a]
xargs: command line cannot be assembled, too long
exit 1
```

`-0` doesn't change this limit. `-S` raises it:

```console
$ tr '\n' '\0' < mixed.txt | xargs -0 -I {} echo "[{}]" > out0.txt; echo "exit $?"
xargs: command line cannot be assembled, too long
exit 1
$ cat out0.txt
[a]
$ tr '\n' '\0' < mixed.txt | xargs -0 -S 4096 -I {} echo "[{}]" | cut -c1-10
[a]
[xxxxxxxxx
[b]
```

A large value is accepted with a short line:

```console
$ printf 'a\n' | xargs -S 1000000 -I {} echo got {}; echo "exit $?"
got a
exit 0
```

I didn't look for an upper limit beyond that.

## Checklist

- `xargs: invalid option -- d` / `-- i` on macOS: `/usr/bin/xargs` is BSD
  xargs.
- Without `-d`, a space in a name splits it, and an apostrophe stops xargs
  with "unterminated quote".
- One item per line: `tr '\n' '\0' < list | xargs -0 ...` (or
  `find ... -print0 | xargs -0`).
- `-i` becomes `-I {}`. Put `tr '\n' '\0' | xargs -0` in front if leading
  blanks, quotes, or backslashes must survive.
- With `-I`, a replaced argument of 255 bytes or more failed here with
  "command line cannot be assembled, too long". xargs stopped at that line
  and exited 1. Add `-S` with room for your longest line (`-S 4096` here).

*Tested on macOS 26.5.2 (Apple silicon) with `/usr/bin/xargs`
(`PROGRAM:xargs  PROJECT:shell_cmds-329`) in zsh 5.9 (arm64-apple-darwin25.0).
GNU findutils xargs and Linux were not tested.*
