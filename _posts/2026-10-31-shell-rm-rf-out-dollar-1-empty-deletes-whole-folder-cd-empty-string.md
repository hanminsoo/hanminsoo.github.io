---
title: "`rm -rf \"out/$1\"` deleted the whole `out` folder: an empty argument, and why `cd \"\" || exit 1` doesn't stop it (bash, zsh on macOS)"
description: >-
  A script meant to delete one subfolder, `rm -rf "out/$1"`, deletes all of
  `out/` when it's run with no argument or an empty one, and exits 0. A slash
  as the argument does the same. `[ -d "out/$1" ]` passes too. And
  `cd "$1" || exit 1` doesn't catch an empty `$1`: `cd ""` succeeds in bash,
  zsh, sh and dash on macOS and stays in the current folder, so the next
  `rm *.png` deletes there. Unquoted `cd $1` goes to your home folder. Fix with
  `${1:?}` plus a check that the name has no slash and isn't `.` or `..`.
  Reproduced in a scratch folder on macOS bash 3.2 and zsh 5.9.
tags: [shell, bash, zsh, macos]
---

A script is supposed to delete one run's output, `out/run1`, and leave the
rest of `out/` alone:

```bash
#!/bin/bash
# clear-run.sh -- delete one run's folder: bash clear-run.sh run1
rm -rf "out/$1"
```

Run it without an argument, or with an empty one from a variable that didn't
get filled in, and `"out/$1"` becomes `"out/"`. rm then deletes the whole
`out` folder, every run in it, and the script exits 0. No root folder, no
glob, nothing that looks dangerous in the source.

A related trap: `cd "$1" || exit 1`, the usual guard for a failed `cd`,
doesn't fire for an empty `$1`. On this Mac, `cd ""` succeeds and stays where
you are, so whatever the script deletes next is deleted in the folder you ran
it from.

The [earlier post on empty variables in `rm`]({% post_url 2026-10-20-shell-rm-empty-variable-deletes-from-root-set-u-colon-question %})
covers `"$DIR"/*` expanding to the top of the disk and why `set -u` misses
empty values. This one is about the quieter cases: a path with a fixed prefix,
and `cd` with an empty argument.

**The `rm` calls below are real. They run only inside a new, empty `demo`
folder. If you try them, do the same.**

## Setup

```console
$ mkdir demo && cd demo
$ mkdir -p out/run1 out/run2 photos fakehome
$ touch out/run1/a.png out/run2/b.png out/notes.txt cover.png photos/p1.png fakehome/avatar.png
```

## `rm -rf "out/$1"` with no argument deletes `out/`

With an argument, it does what it should:

```console
$ bash clear-run.sh run1; echo "exit $?"
exit 0
$ find out | sort
out
out/notes.txt
out/run2
out/run2/b.png
```

Without one:

```console
$ bash clear-run.sh; echo "exit $?"
exit 0
$ ls out
ls: out: No such file or directory
```

`out/run2` and `out/notes.txt` are gone with it, and the exit status is 0.
`bash clear-run.sh ""` does the same, which is what happens when the caller
passes `"$RUN"` and `RUN` is empty.

A `/` as the argument gives `out//`, which is also just `out`:

```console
$ mkdir -p out/run2 && touch out/run2/b.png
$ bash clear-run.sh /; echo "exit $?"
exit 0
$ ls out
ls: out: No such file or directory
```

`.` and `..` are the cases macOS rm does refuse, even with a prefix in front:

```console
$ mkdir -p out/run2 && touch out/run2/b.png
$ bash clear-run.sh .; echo "exit $?"
rm: "." and ".." may not be removed
exit 1
```

(I didn't run `bash clear-run.sh ..` in this post's replay, since `out/..` is
the `demo` folder itself. In my first test, in another scratch folder, it was
refused with the same message.)

### `[ -d "out/$1" ]` doesn't help

A check that the folder exists passes, because `out/` is a folder:

```console
$ bash -c '[ -d "out/$1" ] && echo "out/$1 is a folder"' _ ""
out/ is a folder
```

`set -u` only half helps. With no argument, `$1` is unset and `set -u`
stops the script. With `""`, `$1` is set, just empty, and it goes through:

```console
$ bash -c 'set -u; echo "out/$1"; echo after' _ ""; echo "exit $?"
out/
after
exit 0
```

`${1:?}` catches both.

## `cd ""` succeeds, so `cd "$1" || exit 1` carries on

On macOS, every shell I tried treats an empty `cd` argument as "stay here"
and returns 0:

```console
$ for sh in bash zsh sh dash; do $sh -c 'cd ""; echo "$0: exit $?"' $sh; done
bash: exit 0
zsh: exit 0
sh: exit 0
dash: exit 0
```

So this script, which looks guarded, deletes in the wrong folder:

```bash
#!/bin/bash
# clean-in.sh -- delete the PNGs in the folder given as $1
cd "$1" || exit 1
rm -f *.png
```

```console
$ find . -name '*.png' | sort
./cover.png
./fakehome/avatar.png
./out/run2/b.png
./photos/p1.png
$ bash clean-in.sh ""; echo "exit $?"
exit 0
$ find . -name '*.png' | sort
./fakehome/avatar.png
./out/run2/b.png
./photos/p1.png
```

`cover.png`, in the folder I ran the script from, is gone. Exit 0, no message.

### Unquoted `cd $1` goes to your home folder

Without quotes, an empty `$1` disappears from the command line, and a bare
`cd` goes to `$HOME`. This script only prints where each `cd` lands. I set
`HOME` to the scratch `fakehome` folder for the run, so nothing real is
involved:

```bash
#!/bin/bash
# where.sh -- only prints where cd ends up
cd "$1" || exit 1
echo "quoted:   ${PWD##*/}"
cd "$OLDPWD"
cd $1 || exit 1
echo "unquoted: ${PWD##*/}"
```

```console
$ HOME=$PWD/fakehome bash where.sh ""; echo "exit $?"
quoted:   demo
unquoted: fakehome
exit 0
```

With a real `HOME`, the `rm -f *.png` after an unquoted `cd $1` would run in
your home folder. I didn't run that.

## The fixed scripts

```bash
#!/bin/bash
# clear-run-fixed.sh -- delete one run's folder: bash clear-run-fixed.sh run1
set -u
run=${1:?usage: clear-run-fixed.sh RUN_NAME}
case $run in
  */*|.|..) echo "bad run name: $run" >&2; exit 1 ;;
esac
rm -rf -- "out/$run"
```

- `${1:?...}` stops on a missing **or empty** argument, before rm.
- The `case` refuses anything with a slash (`/`, `../demo`, `run1/../..`)
  and `.` / `..`, so the path can only name something directly inside `out/`.
- `--` keeps a name starting with `-` from being read as an rm option.

```console
$ mkdir -p out/run1 out/run2 && touch out/run1/a.png out/run2/b.png out/notes.txt
$ bash clear-run-fixed.sh; echo "exit $?"
clear-run-fixed.sh: line 4: 1: usage: clear-run-fixed.sh RUN_NAME
exit 1
$ bash clear-run-fixed.sh ""; echo "exit $?"
clear-run-fixed.sh: line 4: 1: usage: clear-run-fixed.sh RUN_NAME
exit 1
$ bash clear-run-fixed.sh /; echo "exit $?"
bad run name: /
exit 1
$ bash clear-run-fixed.sh ..; echo "exit $?"
bad run name: ..
exit 1
$ bash clear-run-fixed.sh run1/../..; echo "exit $?"
bad run name: run1/../..
exit 1
$ bash clear-run-fixed.sh run1; echo "exit $?"
exit 0
$ find out | sort
out
out/notes.txt
out/run2
out/run2/b.png
```

For the `cd` script, refuse the empty argument before `cd`:

```bash
#!/bin/bash
# clean-in-fixed.sh -- delete the PNGs in the folder given as $1
dir=${1:?usage: clean-in-fixed.sh DIR}
cd -- "$dir" || exit 1
rm -f -- *.png
```

```console
$ touch cover.png
$ bash clean-in-fixed.sh ""; echo "exit $?"
clean-in-fixed.sh: line 3: 1: usage: clean-in-fixed.sh DIR
exit 1
$ bash clean-in-fixed.sh photoz; echo "exit $?"
clean-in-fixed.sh: line 4: cd: photoz: No such file or directory
exit 1
$ bash clean-in-fixed.sh photos; echo "exit $?"
exit 0
$ find . -name '*.png' | sort
./cover.png
./fakehome/avatar.png
./out/run2/b.png
```

Only `photos/p1.png` was deleted. (The `1:` in bash's message is the name of
the parameter, `$1`.)

## Checklist

- `rm -rf "prefix/$1"` with an empty or missing `$1` deletes all of
  `prefix/`, exit 0. So does `$1=/`.
- macOS rm refuses `prefix/.` and `prefix/..`, but not `prefix/` or
  `prefix//`.
- `[ -d "prefix/$1" ]` passes when `$1` is empty, because `prefix/` is a
  folder.
- `cd ""` returns 0 and stays in the current folder (bash, zsh, sh, dash on
  macOS), so `cd "$1" || exit 1` doesn't catch an empty `$1`.
- Unquoted `cd $1` with an empty `$1` goes to `$HOME`.
- Use `${1:?message}` before building the path or calling `cd`, and refuse
  names with `/`, `.` and `..` if the argument should name one entry.
- Test destructive scripts with no argument and with `""` in a scratch
  folder before pointing them at real data.

*Tested on macOS 26.5.2 (Apple silicon) with `/bin/bash` (GNU bash, version
3.2.57(1)-release (arm64-apple-darwin25)), `/bin/zsh` (zsh 5.9
(arm64-apple-darwin25.0)), `/bin/sh` and `/bin/dash`, macOS `/bin/rm`. All
real `rm` calls ran in a scratch folder under `/tmp`; the home-folder case was
run with `HOME` pointed at a scratch folder and only printed the folder.
Linux, newer bash versions and other `cd` implementations were not tested.*
