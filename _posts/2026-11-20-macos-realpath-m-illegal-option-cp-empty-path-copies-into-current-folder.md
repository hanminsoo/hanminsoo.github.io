---
title: "macOS `realpath -m`: \"illegal option -- m\" — and the backup script that copies into the current folder"
description: >-
  `realpath -m path` fails on a Mac with "realpath: illegal option -- m" and
  exit 1, and plain `realpath` fails on any path that does not exist yet,
  even if only the last part is missing. In a script `dest=$(realpath -m
  "$1")` is empty, `mkdir -p ""` fails, and `cp -R notes/ ""` copies into the
  current folder, overwriting same-named files, exit 0. Fix: create the folder
  first and then call `realpath`, check every `$(...)`, or use Python's
  `os.path.realpath` for deep missing paths.
tags: [macos, realpath, shell, bsd]
---

Turning a path that does not exist yet into an absolute path, on a Mac:

```console
$ mkdir /tmp/rpdemo && cd /tmp/rpdemo
$ realpath -m backup/2026-11-20; echo "exit $?"
realpath: illegal option -- m
usage: realpath [-q] [path ...]
exit 1
$ realpath backup/2026-11-20; echo "exit $?"
realpath: backup/2026-11-20: No such file or directory
exit 1
```

GNU `realpath -m` ("canonicalize missing") accepts a path where any part may
be missing, and GNU `realpath` without options accepts a missing last part.
That is how the GNU manual describes them; I did not run GNU coreutils for
this post. The `realpath` that ships with macOS (`/bin/realpath`) has one
option, `-q`, and its man page says it just calls `realpath(3)`. Every part of
the path has to exist.

On a terminal you see the error. In a script it turns into an empty string,
and the next commands do something you did not ask for.

## The backup script that copies into the current folder

A small script that copies `notes/` into a dated folder and prints the
absolute path:

```console
$ mkdir notes
$ printf 'hello\n' > notes/a.txt
$ cat save.sh
#!/bin/sh
# save.sh DEST - copy notes/ into DEST and say where it went
dest=$(realpath -m "$1")
mkdir -p "$dest"
cp -R notes/ "$dest"
echo "saved to $dest"
$ ls
notes
save.sh
$ sh save.sh backup/2026-11-20; echo "exit $?"
realpath: illegal option -- m
usage: realpath [-q] [path ...]
mkdir: : No such file or directory
saved to 
exit 0
$ ls
a.txt
notes
save.sh
$ ls backup
ls: backup: No such file or directory
$ rm a.txt
```

No `backup` folder, a stray `a.txt` next to the script, "saved to " with
nothing after it, and exit 0. Step by step:

- `realpath` writes its error to stderr and nothing to stdout, so `dest` is
  empty. The assignment's status is 1, but nothing checks it:

```console
$ dest=$(realpath -m backup 2>/dev/null); echo "status $? dest=[$dest]"
status 1 dest=[]
```

- `mkdir -p ""` fails (exit 1), and the script goes on:

```console
$ mkdir -p ""; echo "exit $?"
mkdir: : No such file or directory
exit 1
```

- `cp` with an empty target does not fail. On this Mac it copied into the
  current folder and exited 0, both for `cp -R folder/` and for a single file.
  (The man page does not say what an empty target means; this is what I saw.)

```console
$ cp -R notes/ ""; echo "exit $?"
exit 0
$ ls
a.txt
notes
save.sh
$ rm a.txt
$ cp notes/a.txt ""; echo "exit $?"
exit 0
$ ls
a.txt
notes
save.sh
$ rm a.txt
```

"The current folder" is wherever the script was started from, and `cp`
overwrites files there without asking. With an `a.txt` of its own in the
folder, and stderr hidden as it would be in a cron or launchd log nobody
reads:

```console
$ printf 'keep me\n' > a.txt
$ sh save.sh backup/2026-11-20 2>/dev/null; echo "exit $?"
saved to 
exit 0
$ cat a.txt
hello
$ rm a.txt
```

`keep me` is gone.

## Plain `realpath` needs the whole path to exist

Dropping `-m` does not help when the path is new, even if only the last part
is missing. `-q` only hides the message; the status is still 1 and the output
still empty:

```console
$ realpath notes; echo "exit $?"
/private/tmp/rpdemo/notes
exit 0
$ realpath notes/new.txt; echo "exit $?"
realpath: notes/new.txt: No such file or directory
exit 1
$ realpath -q notes/new.txt; echo "exit $?"
exit 1
```

(The `/private` prefix is real: on macOS `/tmp` is a symlink to
`/private/tmp`, and `realpath` resolves symlinks.)

The other GNU options fail the same way, each with exit 1:

```console
$ for o in -e -s -L -P -z --relative-to=.; do realpath $o notes; echo "exit $?"; done
realpath: illegal option -- e
usage: realpath [-q] [path ...]
exit 1
realpath: illegal option -- s
usage: realpath [-q] [path ...]
exit 1
realpath: illegal option -- L
usage: realpath [-q] [path ...]
exit 1
realpath: illegal option -- P
usage: realpath [-q] [path ...]
exit 1
realpath: illegal option -- z
usage: realpath [-q] [path ...]
exit 1
realpath: illegal option -- -
usage: realpath [-q] [path ...]
exit 1
```

## Fix 1: create it first, then resolve it

If the script is going to create the folder anyway, create it first. Then it
exists and plain `realpath` works. Check every step:

```console
$ cat save.sh
#!/bin/sh
# save.sh DEST - copy notes/ into DEST and say where it went
dest=${1:?usage: save.sh DEST}
mkdir -p "$dest" || exit 1
dest=$(realpath "$dest") || exit 1
cp -R notes/ "$dest" || exit 1
echo "saved to $dest"
$ sh save.sh backup/2026-11-20; echo "exit $?"
saved to /private/tmp/rpdemo/backup/2026-11-20
exit 0
$ ls backup/2026-11-20
a.txt
$ sh save.sh notes/a.txt/old; echo "exit $?"
mkdir: notes/a.txt: Not a directory
exit 1
$ sh save.sh; echo "exit $?"
save.sh: line 3: 1: usage: save.sh DEST
exit 1
$ ls
backup
notes
save.sh
```

`dest=$(realpath "$dest") || exit 1` works because a plain assignment takes
the status of its command substitution. `${1:?...}` stops the script when the
argument is missing or empty, before anything else runs.

## Fix 2: a file that does not exist yet — resolve its folder

For an output file you have not written yet, resolve the folder and add the
name. Do it in two steps. On one line, the check is lost:

```console
$ f=out/report.txt
$ abs=$(realpath "$(dirname "$f")")/$(basename "$f"); echo "exit $? abs=$abs"
realpath: out: No such file or directory
exit 0 abs=/report.txt
```

With two command substitutions in one assignment, the status is that of the
last one (`basename`, 0), and the result is `/report.txt`, at the root of the
disk. Split it so the `realpath` status is the one you check:

```console
$ dir=$(realpath "$(dirname "$f")") || echo "stop: no parent"
realpath: out: No such file or directory
stop: no parent
$ mkdir out
$ dir=$(realpath "$(dirname "$f")") && abs=$dir/$(basename "$f"); echo "exit $? abs=$abs"
exit 0 abs=/private/tmp/rpdemo/out/report.txt
```

In a script, write the first line as `dir=$(realpath "$(dirname "$f")") || exit 1`.

## Fix 3: several missing levels — Python's `os.path.realpath`

If you really need `-m` (an absolute path for something several levels deep
that you do not want to create yet), Python's `os.path.realpath` does not
require the path to exist. On this Mac `/usr/bin/python3` is 3.8.2:

```console
$ realpath_m() { [ -n "$1" ] || { echo "realpath_m: empty path" >&2; return 1; }; python3 -c 'import os, sys; print(os.path.realpath(sys.argv[1]))' "$1"; }
$ realpath_m backup/2027/01/01
/private/tmp/rpdemo/backup/2027/01/01
$ realpath_m ""; echo "exit $?"
realpath_m: empty path
exit 1
```

The empty check is there because Python turns an empty path into the current
folder, the same trap as the `cp` above:

```console
$ python3 -c 'import os; print(os.path.realpath(""))'
/private/tmp/rpdemo
```

In zsh, the `${p:A}` modifier also accepts missing paths, but it is not the
same as `realpath` when `..` follows a symlink. Here `d` points into
`elsewhere/deep`:

```console
$ mkdir -p elsewhere/deep
$ ln -s elsewhere/deep d
$ realpath d/..
/private/tmp/rpdemo/elsewhere
$ realpath_m d/../new.txt
/private/tmp/rpdemo/elsewhere/new.txt
$ p=d/../new.txt; print -r -- ${p:A}
/private/tmp/rpdemo/new.txt
```

`realpath` and Python follow the symlink first and then go up from
`elsewhere/deep`. zsh removes `d/..` as text first and never looks at the
symlink. If your paths can contain `..`, the two give different folders.

Clean up:

```console
$ cd / && rm -rf /tmp/rpdemo
```

## Checklist

- macOS `/bin/realpath` has only `-q`. `-m`, `-e`, `-s`, `-L`, `-P`, `-z` and
  `--relative-to` are "illegal option", exit 1.
- Without options it fails (exit 1, "No such file or directory") if any part
  of the path is missing, the last part included. `-q` only hides the message.
- In a script the failure becomes an empty string. `mkdir -p ""` fails, but
  `cp -R dir/ ""` and `cp file ""` copied into the current folder and
  overwrote a same-named file, exit 0 (seen on this Mac).
- Folder you will create anyway: `mkdir -p "$d" || exit 1`, then
  `d=$(realpath "$d") || exit 1`.
- File not written yet: resolve its folder in its own assignment and check
  it. `x=$(realpath ...)/$(basename ...)` takes `basename`'s status and can
  give `/name`.
- Deep missing paths: `python3 -c 'import os, sys; print(os.path.realpath(sys.argv[1]))'`,
  after checking the argument is not empty. zsh `${p:A}` gives a different
  answer for `symlink/..`.

Related: [empty variable in a path: `rm -rf "out/$1"`, `cd ""`](/2026/10/shell-rm-rf-out-dollar-1-empty-deletes-whole-folder-cd-empty-string/)
and [macOS `cp -R folder/` copies the contents](/2026/10/macos-cp-r-trailing-slash-copies-contents-not-folder/).

*Tested on 2026-11-20 on macOS 26.5.2 (Apple silicon, APFS) with `/bin/realpath`
(shell_cmds-329), `/bin/cp` and `/bin/mkdir` (file_cmds-479),
`/usr/bin/python3` 3.8.2, zsh 5.9 and `/bin/sh` (bash 3.2.57), in `env -i`
with `PATH=/usr/bin:/bin:/usr/sbin:/sbin`.
Not tested: GNU coreutils `realpath`, Linux, other Python versions, other
filesystems, older macOS versions.*
