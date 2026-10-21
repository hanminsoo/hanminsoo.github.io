---
title: "macOS `cp --parents`: \"cp: illegal option -- -\". Copy files with their folder path using `pax -rw` or `rsync -R` (and openrsync ignores `/./`)"
description: >-
  `cp --parents src/a/b.png out/` fails on macOS with "cp: illegal option -- -"
  (exit 64), because /bin/cp is BSD cp. `cp -R` and `ditto` copy the file but
  drop its folder path. `pax -rw FILES DEST` keeps the path, reads a file list
  from `find` on stdin, handles spaces, and keeps mtimes, but DEST must already
  exist. `rsync -R` also keeps the path, but the rsync that ships with macOS is
  openrsync, which ignored the `/./` marker in my test; `cd` first instead.
  Also: `du --max-depth` is `du -d`. Reproduced on macOS 26.
tags: [macos, shell, cp, rsync, bsd]
---

You want to copy a few files into another folder and keep their
relative paths, `src/app/img/logo.png` landing at
`out/src/app/img/logo.png`. On Linux that's `cp --parents`. On a Mac:

```console
$ mkdir demo && cd demo
$ export TZ=UTC
$ mkdir -p src/app/img src/lib 'src/my dir'
$ echo a > src/app/img/logo.png; echo b > src/lib/util.js; echo c > src/app/main.js; echo d > 'src/my dir/x y.js'
$ mkdir out && cp --parents src/app/img/logo.png src/lib/util.js out/; echo "exit $?"
cp: illegal option -- -
usage: cp [-R [-H | -L | -P]] [-fi | -n] [-aclpSsvXx] source_file target_file
       cp [-R [-H | -L | -P]] [-fi | -n] [-aclpSsvXx] source_file ... target_directory
exit 64
$ what /bin/cp /bin/pax
/bin/cp:
	PROGRAM:cp  PROJECT:file_cmds-479
	PROGRAM:cp  PROJECT:file_cmds-479
/bin/pax:
	PROGRAM:pax  PROJECT:file_cmds-479
	PROGRAM:pax  PROJECT:file_cmds-479
```

BSD cp has no `--parents`. Below: two built-in tools that keep the path (`pax`
and `rsync -R`), what to watch for with each, and the same kind of error
from `du --max-depth`.

## `cp -R` and `ditto` drop the path

```console
$ cp -R src/app/img/logo.png out/; find out | sort
out
out/logo.png
$ ditto src/app/img/logo.png out3/; find out3 | sort
out3
out3/logo.png
```

Both copy the file to the top of the target, which is what `--parents`
exists to avoid.

## `pax -rw`: copies the path as given

`pax` ships with macOS. With `-rw` it reads and writes in one step, copying
each operand under the target folder **with the path you wrote**:

```console
$ mkdir out4 && pax -rw src/app/img/logo.png src/lib/util.js out4; echo "exit $?"
exit 0
$ find out4 | sort
out4
out4/src
out4/src/app
out4/src/app/img
out4/src/app/img/logo.png
out4/src/lib
out4/src/lib/util.js
```

The usual `find ... -exec cp --parents {} dest \;` becomes a pipe. With no
file operands, pax reads the list from stdin, and names with spaces come
through:

```console
$ mkdir out8 && find src -name '*.js' | pax -rw out8; echo "exit $?"
exit 0
$ find out8 | sort
out8
out8/src
out8/src/app
out8/src/app/main.js
out8/src/lib
out8/src/lib/util.js
out8/src/my dir
out8/src/my dir/x y.js
```

To leave out the leading `src/`, run it from inside:

```console
$ mkdir out9 && (cd src && find . -name '*.js' | pax -rw ../out9); find out9 | sort
out9
out9/app
out9/app/main.js
out9/lib
out9/lib/util.js
out9/my dir
out9/my dir/x y.js
```

The target has to exist. pax doesn't create it:

```console
$ pax -rw src/lib/util.js out7; echo "exit $?"
pax: Cannot access destination directory out7 <No such file or directory>
exit 1
```

The list is one name per line, so a file name containing a newline would
break it. I didn't test that case.

## `rsync -R`: also keeps the path, but `/./` was ignored

```console
$ rsync --version | head -2
openrsync: protocol version 29
rsync version 2.6.9 compatible
$ rsync -R src/app/img/logo.png src/lib/util.js out1/; echo "exit $?"
exit 0
$ find out1 | sort
out1
out1/src
out1/src/app
out1/src/app/img
out1/src/app/img/logo.png
out1/src/lib
out1/src/lib/util.js
```

The `/usr/bin/rsync` on this Mac is **openrsync**, not Samba rsync. With
Samba rsync, the documentation says a `/./` in the path marks where the
copied path should start, so `src/./app/img/logo.png` would land as
`app/img/logo.png`. Here it was ignored, and the whole path was copied:

```console
$ rsync -R src/./app/img/logo.png out2/; find out2 | sort
out2
out2/src
out2/src/app
out2/src/app/img
out2/src/app/img/logo.png
```

I didn't run Samba rsync to compare. To get the shorter path with
openrsync, `cd` first:

```console
$ (cd src && rsync -R app/img/logo.png ../out6/); find out6 | sort
out6
out6/app
out6/app/img
out6/app/img/logo.png
```

## Modification times

`pax -rw` kept the original mtime. `rsync -R` without `-t` gave the copy
the current time, and `-t` kept it:

```console
$ touch -t 202001010000 src/lib/util.js
$ mkdir outA && pax -rw src/lib/util.js outA
$ rsync -R src/lib/util.js outB/; rsync -Rt src/lib/util.js outC/
$ stat -f '%Sm %N' src/lib/util.js outA/src/lib/util.js outB/src/lib/util.js outC/src/lib/util.js
Jan  1 00:00:00 2020 src/lib/util.js
Jan  1 00:00:00 2020 outA/src/lib/util.js
Oct 21 01:00:07 2026 outB/src/lib/util.js
Jan  1 00:00:00 2020 outC/src/lib/util.js
```

(The `outB` line shows the time I ran it.)

## Same kind of error: `du --max-depth`

```console
$ du --max-depth=1 src; echo "exit $?"
du: unrecognized option `--max-depth=1'
usage: du [-Aclnx] [-H | -L | -P] [-g | -h | -k | -m] [-a | -s | -d depth] [-B blocksize] [-I mask] [-t threshold] [file ...]
exit 64
$ du -d 1 src | sort -k2
32	src
16	src/app
8	src/lib
8	src/my dir
```

BSD du spells it `-d 1`.

## Checklist

- `cp: illegal option -- -` (exit 64): macOS `/bin/cp` is BSD cp, with no
  `--parents`.
- `cp -R file dest/` and `ditto file dest/` drop the folder path.
- `pax -rw FILES DEST` keeps the path as written. `find ... | pax -rw DEST`
  replaces `find -exec cp --parents`. DEST must exist. Spaces are fine.
  It keeps mtimes.
- `rsync -R FILES DEST/` keeps the path. Add `-t` to keep mtimes. macOS's
  rsync is openrsync (2.6.9 compatible), and `/./` was ignored here, so `cd`
  into the base folder first.
- To drop a leading folder, run from inside it: `(cd src && ... ../DEST)`.
- `du --max-depth=1` is `du -d 1` on macOS.

*Tested on macOS 26.5.2 (Apple silicon) with `/bin/cp` and `/bin/pax`
(`file_cmds-479`), `/usr/bin/rsync` (`openrsync: protocol version 29`,
`rsync version 2.6.9 compatible`), `/usr/bin/du` (`file_cmds-479`), in zsh 5.9
(arm64-apple-darwin25.0), `TZ=UTC`. GNU cp, Samba rsync and Linux were not
tested.*
