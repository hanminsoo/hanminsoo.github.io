---
title: "`rm -rf \"$DIR\"/*` with an empty variable expands to everything under `/`: why `set -u` doesn't stop it and `${DIR:?}` does (bash, zsh on macOS)"
description: >-
  A cleanup line like `rm -f "$OUT_DIR"/*.png` turns into `rm -f /*.png`
  when the variable is empty or misspelled, and `rm -rf "$OUT_DIR"/*` turns into
  `rm -rf /Applications /bin ... /Users ...`, every top-level item of the disk.
  (macOS rm refuses `/` itself, but not the items under it.) `set -u` catches a misspelled (unset) name but not a variable that
  is set to an empty string, for example from a failed `$(...)`. `${OUT_DIR:?}`
  catches both and stops the script before rm runs. Also: `cd "$1"; rm *.png`
  deletes in the wrong folder when cd fails. Reproduced with echo dry runs and
  a scratch folder on macOS bash 3.2 and zsh 5.9.
tags: [shell, bash, zsh, macos]
---

A cleanup script ends with a line like this. **Don't copy and run it.**
It's the bug this post is about:

```bash
echo rm -f "$OUT_DIR"/*.png    # echo added: prints the command, deletes nothing
```

If `OUT_DIR` is empty, misspelled, or was never set, the shell doesn't
complain. It quietly expands the line to `rm -f /*.png`. With
`rm -rf "$OUT_DIR"/*` ("empty the output folder"), it expands to every item
at the top of the disk. Below, I show what the shell builds in
each case, why `set -u` only catches half of them, and the one-character
guard (`${OUT_DIR:?}`) that catches both. I also show a related bug where a
failed `cd` makes rm delete files in the wrong folder.

**Everything that would touch `/` is shown with `echo` in front**, so the
shell prints the command instead of running it. The only real `rm` calls in
this post run inside a throwaway `demo` folder. Don't drop the `echo` to "see
what happens".

## Setup

Everything below runs in one empty folder:

```console
$ mkdir demo && cd demo
$ mkdir out && touch out/frame1.png out/frame2.png cover.png
```

## What the shell builds from an empty variable

Case 1, a typo. The script sets `OUT_DIR` and then uses `OUTDIR`:

```bash
#!/bin/bash
# dry-typo.sh -- echo prints the command instead of running it
OUT_DIR=out
echo rm -f "$OUTDIR"/*.png
```

Case 2, a variable that is set, but to an empty string. Here the folder name
comes from a file that doesn't exist, so `$(...)` gives nothing:

```bash
#!/bin/bash
# dry-empty.sh
OUT_DIR=$(cat outdir.txt 2>/dev/null)
echo rm -rf "$OUT_DIR"/*
```

```console
$ bash dry-typo.sh; echo "exit $?"
rm -f /*.png
exit 0
$ bash dry-empty.sh; echo "exit $?"
rm -rf /Applications /bin /cores /dev /etc /home /Library /opt /private /sbin /System /tmp /Users /usr /var /Volumes
exit 0
```

Both scripts "succeed". Without the `echo`, the first one would try to
delete PNG files at the top of the disk. The second one would hand rm
every top-level item of the disk, `/Users` included. I didn't run either
one, so I can't tell you how far it would get without sudo.

You may have heard that `rm -rf /` itself is refused. On macOS it is; the
rm man page says:

> It is an error to attempt to remove the files /, . or ..

But that check is about the argument `/`. With `"$OUT_DIR"/*`, the shell
expands the glob first, so rm never sees `/`. It gets `/Applications`,
`/Users`, and the rest as separate arguments. (I took the sentence from
`man rm` and didn't run `rm -rf /` to check it.)

(The `/*.png` is a glob. If the root folder had PNG files, they would
appear in the printed command. On my Mac it has none, so bash leaves the
pattern as it is. The long list in the second output is what `/*` matches on
my Mac; yours may differ.)

## `set -u` catches the typo, not the empty string

`set -u` (`nounset`) makes the shell stop when it expands a variable that
was **never set**. A variable that is set to `""` is not "unset," so it goes
through:

```bash
#!/bin/bash
# u-typo.sh
set -u
OUT_DIR=out
echo rm -f "$OUTDIR"/*.png
echo "still running"
```

```bash
#!/bin/bash
# u-empty.sh
set -u
OUT_DIR=$(cat outdir.txt 2>/dev/null)
echo rm -rf "$OUT_DIR"/*
echo "still running"
```

```console
$ bash u-typo.sh; echo "exit $?"
u-typo.sh: line 5: OUTDIR: unbound variable
exit 1
$ bash u-empty.sh; echo "exit $?"
rm -rf /Applications /bin /cores /dev /etc /home /Library /opt /private /sbin /System /tmp /Users /usr /var /Volumes
still running
exit 0
```

The empty case is the one you get from a failed command substitution, an
empty line in a config file, or an environment variable that exists but is
blank. `set -u` lets all of these through.

## `${VAR:?}` stops on both

`${OUT_DIR:?}` expands to the value, unless the variable is unset **or
empty**. Then the shell prints an error, and a script stops right there,
before the command runs. You can add your own message after the `?`:

```bash
#!/bin/bash
# q-typo.sh
OUT_DIR=out
echo rm -f "${OUTDIR:?}"/*.png
echo "still running"
```

```bash
#!/bin/bash
# q-empty.sh
OUT_DIR=$(cat outdir.txt 2>/dev/null)
echo rm -rf "${OUT_DIR:?output folder not set}"/*
echo "still running"
```

```bash
#!/bin/bash
# q-nocolon.sh -- without the colon, only "unset" is caught
OUT_DIR=$(cat outdir.txt 2>/dev/null)
echo rm -rf "${OUT_DIR?}"/*
echo "still running"
```

```console
$ bash q-typo.sh; echo "exit $?"
q-typo.sh: line 4: OUTDIR: parameter null or not set
exit 1
$ bash q-empty.sh; echo "exit $?"
q-empty.sh: line 4: OUT_DIR: output folder not set
exit 1
$ bash q-nocolon.sh; echo "exit $?"
rm -rf /Applications /bin /cores /dev /etc /home /Library /opt /private /sbin /System /tmp /Users /usr /var /Volumes
still running
exit 0
```

The colon matters. `${VAR?}` behaves like `set -u` and lets an empty value
through. `${VAR:?}` doesn't.

On exit codes: run from a script file, these errors exit with 1. In my
tests, the same expansions under `bash -c '...'` exited with 127 instead.
Either way, it's non-zero and nothing after it runs. I didn't track down why
the two codes differ.

### zsh

The default shell on macOS is zsh. `${VAR:?}` works the same way there, and
the message says "parameter not set" even when the variable is set but empty:

```console
$ zsh -c 'D=""; echo rm -rf "${D:?}"/*; echo "still running"'; echo "exit $?"
zsh:1: D: parameter not set
exit 1
$ zsh -c 'D=""; echo rm -rf "$D"/*; echo "still running"'; echo "exit $?"
rm -rf /Applications /bin /cores /dev /etc /home /Library /opt /private /sbin /System /tmp /Users /usr /var /Volumes
still running
exit 0
```

zsh does refuse `/*.png` when nothing matches, because a glob that matches
nothing is an error there by default:

```console
$ zsh -c 'D=""; echo rm -f "$D"/*.png; echo "still running"'; echo "exit $?"
zsh:1: no matches found: /*.png
exit 1
```

That's an accident of the glob, though. `"$D"/*` always matches something,
so it doesn't protect `rm -rf "$D"/*`.

## A related one: `cd` fails, rm runs in the wrong folder

This one doesn't need an empty variable. The script trusts that `cd`
worked. **It really deletes files. If you want to try it, run it only in a new,
empty test folder like `demo` above, never in a folder with files you care
about:**

```bash
#!/bin/bash
# clean-in.sh -- delete PNGs in the folder given as $1
cd "$1"
rm -f *.png
```

With a typo in the folder name, `cd` fails, the script carries on, and
`*.png` matches the files in the folder you ran it from. This deletion is
real, so it runs in `demo`:

```console
$ find . -name '*.png' | sort
./cover.png
./out/frame1.png
./out/frame2.png
$ bash clean-in.sh ouy; echo "exit $?"
clean-in.sh: line 3: cd: ouy: No such file or directory
exit 0
$ find . -name '*.png' | sort
./out/frame1.png
./out/frame2.png
```

`cover.png`, which was not in `out`, is gone. The files that should have
been deleted are still there, and the script exited 0.

## The fixed script

```bash
#!/bin/bash
# clean.sh -- delete PNGs in OUT_DIR
set -u
OUT_DIR=${1:?usage: clean.sh OUT_DIR}
[ -d "$OUT_DIR" ] || { echo "no such folder: $OUT_DIR" >&2; exit 1; }
rm -f -- "$OUT_DIR"/*.png
```

- `${1:?...}` refuses a missing **or empty** argument.
- `[ -d ... ] || exit 1` refuses a folder that doesn't exist, instead of
  carrying on somewhere else.
- `rm` gets the full path, so it doesn't depend on the current folder, and
  `--` keeps a name that starts with `-` from being read as an option.
- `set -u` stays on for typos elsewhere in the script.

```console
$ touch cover.png
$ bash clean.sh ouy; echo "exit $?"
no such folder: ouy
exit 1
$ bash clean.sh out; echo "exit $?"
exit 0
$ find . -name '*.png' | sort
./cover.png
```

I didn't run `clean.sh` with no argument or an empty one, because that run
has a real `rm` in it. Its first line is the same `${...:?}` expansion as in
`q-empty.sh` above, which stopped before the next line ran.

If a script has to use `cd`, write `cd "$1" || exit 1`.

### A bash 3.2 catch with `set -u`

The `/bin/bash` that ships with macOS is 3.2. In bash 3.2, `set -u` also
treats an **empty array** as unbound, so turning it on can break a script
that worked before:

```console
$ bash -c 'set -u; a=(); echo "n=${#a[@]}"; echo "${a[@]}"; echo after'; echo "exit $?"
n=0
bash: a[@]: unbound variable
exit 127
$ bash -c 'set -u; a=(); echo ${a[@]+"${a[@]}"}; echo after'; echo "exit $?"

after
exit 0
```

`${a[@]+"${a[@]}"}` expands to the array only if it is set, so it works
under `set -u` on 3.2. I didn't test newer bash versions here.

## Checklist

- `rm ... "$VAR"/...` with an unset or empty `VAR` turns into a path from
  `/`. `rm -rf "$VAR"/*` becomes every top-level item of the disk.
- macOS rm refuses `/` itself (man page), but a glob like `/*` reaches rm as
  separate paths, so that check doesn't help.
- To see what a line will actually run, put `echo` in front of it. Don't
  test it for real.
- `set -u` stops on **unset** variables (typos) but lets **empty** ones
  through (failed `$(...)`, blank config values).
- `${VAR:?message}` stops on both, before the command runs. `${VAR?}`
  without the colon lets empty values through.
- zsh: `${VAR:?}` works the same way. Its "no matches found" for globs
  doesn't protect `rm -rf "$VAR"/*`.
- After `cd`, add `|| exit 1`, or skip `cd` and give rm the full path.
- Check that the folder exists (`[ -d "$DIR" ] || exit 1`) before deleting
  from it.
- bash 3.2 with `set -u`: an empty array is "unbound". Use
  `${a[@]+"${a[@]}"}`.

*Tested on macOS 26.5.2 (Apple silicon) with `/bin/bash` (GNU bash, version
3.2.57(1)-release (arm64-apple-darwin25)) and `/bin/zsh` (zsh 5.9
(arm64-apple-darwin25.0)). Every command that would have touched `/` was
run only with `echo` in front. Real `rm` calls ran only inside a scratch
folder. Linux and newer bash versions were not tested.*
