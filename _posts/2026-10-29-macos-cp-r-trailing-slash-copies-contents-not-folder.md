---
title: "macOS `cp -R folder/ dest/` copies the files, not the folder (exit 0); `cp -R folder dest` nests on the second run; a symlinked folder is copied as a link"
description: >-
  On macOS, `cp -R project/ backups/` spills project's files straight into
  backups/ instead of making backups/project, and exits 0: BSD cp copies the
  contents of a source that ends in a slash. Without the slash, `cp -R project
  backup` makes a copy the first time and backup/project inside it the second
  time. `project/*` skips dotfiles, `project/.` doesn't. A symlink to a folder
  is copied as a symlink (a "backup" that points back at the original, or
  dangles); a trailing slash or `-H` follows it. rsync works the same way
  with the slash; ditto copied contents in my test; mv ignored the slash here. A
  small script that gives the same result with or without the slash.
tags: [macos, shell, cp, rsync, ditto, symlink, backup]
---

A backup step copies a project folder into a folder of backups. Every other
project in there has its own subfolder, but this one doesn't: its files are
loose in `backups/`, mixed with everything else. No error, exit 0:

```console
$ mkdir -p project/sub backups && echo a > project/a.txt && echo b > project/sub/b.txt && echo x > project/.gitignore
$ cp -R project/ backups/; echo "exit $?"
exit 0
$ find backups | sort
backups
backups/.gitignore
backups/a.txt
backups/sub
backups/sub/b.txt
```

The cause is the slash after `project`. This is documented in macOS's
`man cp`, under `-R`:

> If the source_file ends in a /, the contents of the directory are copied
> rather than the directory itself.

So `project/` means "what's inside project". It's easy to get that slash
without meaning to, for example from a variable like `SRC=~/work/project/`
or from a pasted path. Without it, the folder itself goes in:

```console
$ rm -rf backups && mkdir backups
$ cp -R project backups/; find backups | sort
backups
backups/project
backups/project/.gitignore
backups/project/a.txt
backups/project/sub
backups/project/sub/b.txt
```

(I only tested macOS's `/bin/cp` here, not GNU cp.)

## Without the slash, the second run nests

Dropping the slash has its own trap when the destination is the copy
itself, not a folder of copies. `cp -R project backup` gives a different
result depending on whether `backup` already exists. The first run creates
`backup` as the copy. The second run sees an existing folder and copies
`project` *into* it:

```console
$ rm -rf backups
$ cp -R project backup; find backup | sort
backup
backup/.gitignore
backup/a.txt
backup/sub
backup/sub/b.txt
$ cp -R project backup; find backup | sort
backup
backup/.gitignore
backup/a.txt
backup/project
backup/project/.gitignore
backup/project/a.txt
backup/project/sub
backup/project/sub/b.txt
backup/sub
backup/sub/b.txt
```

A nightly job written this way is right on night one. From night two on,
each run refreshes `backup/project`, and the top-level files in `backup`
stay as they were on night one. (Checked separately: both runs exited 0, and
after changing `a.txt` between them, `backup/a.txt` kept the old text while
`backup/project/a.txt` had the new.)

## `folder/*` skips dotfiles; `folder/.` doesn't

If you want the contents, the obvious spelling is `project/*`. The shell
expands `*` before `cp` runs, and `*` doesn't match names that start with a
dot (checked in zsh), so `.gitignore` is left behind.
`project/.` is a path to the folder itself, ending in `.`; `cp -R` copied
its contents, dotfiles included:

```console
$ rm -rf backup && mkdir backup
$ cp -R project/* backup/; find backup | sort
backup
backup/a.txt
backup/sub
backup/sub/b.txt
$ rm -rf backup && mkdir backup
$ cp -R project/. backup/; find backup | sort
backup
backup/.gitignore
backup/a.txt
backup/sub
backup/sub/b.txt
```

`folder/*` also fails outright on an empty folder (`e` here): zsh stops with
`no matches found: e/*`, and bash 3.2 passes the literal `e/*` to cp
(`cp: e/*: No such file or directory`); both exit 1. `project/.` has
neither problem.

## A symlinked folder is copied as a symlink

`man cp` also says `-R` causes "symbolic links to be copied, rather than
indirected through". That includes a link named on the command line. If the
folder you back up is a symlink (`current -> project`, a release link, a
folder you moved and linked back), the "backup" is a new link to the same
place:

```console
$ rm -rf backup && ln -s project current
$ cp -R current backup; stat -f '%N: %HT%SY' backup
backup: Symbolic Link -> project
```

The backup holds no files of its own; it's a pointer to `project`. If the link is relative and
the copy lands in another folder, it doesn't even work:

```console
$ mkdir backups && cp -R current backups/; stat -f '%N: %HT%SY' backups/current
backups/current: Symbolic Link -> project
$ ls backups/current/; echo "exit $?"
ls: backups/current/: No such file or directory
exit 1
```

`backups/current` points at `backups/project`, which doesn't exist. `cp`
printed nothing.

Two ways to make `cp` follow the link. A trailing slash (the path now
goes through the link, and the contents get copied as above), or `-H`,
which follows symlinks named on the command line but not the ones inside
the tree:

```console
$ cp -R current/ backup2; stat -f '%N: %HT%SY' backup2; find backup2 | sort
backup2: Directory
backup2
backup2/.gitignore
backup2/a.txt
backup2/sub
backup2/sub/b.txt
$ rm -rf backup && mkdir backup
$ cp -RH current backup/; stat -f '%N: %HT%SY' backup/current
backup/current: Directory
```

`-L` would follow every symlink inside the tree too, which is a different
copy. I'd use `-H` (or the `/.` form below) for the top-level link only.

## rsync, ditto and mv

Other tools on this Mac, same project, a fresh empty `backups` each time:

```console
$ rm -rf backups && mkdir backups
$ rsync -a project/ backups/; find backups -maxdepth 1 | sort
backups
backups/.gitignore
backups/a.txt
backups/sub
$ rm -rf backups && mkdir backups
$ ditto project backups; find backups -maxdepth 1 | sort
backups
backups/.gitignore
backups/a.txt
backups/sub
$ rm -rf backups && mkdir backups
$ cp -R project moved && mv moved/ backups/; find backups -maxdepth 1 | sort
backups
backups/moved
```

- `rsync` (macOS ships openrsync, "rsync version 2.6.9 compatible") has
  the same rule as `cp`: `project/` copies the contents, `project` the
  folder. On macOS
  cp and rsync agree here.
- `ditto project backups` copied the contents, with no slash at all.
  That's its documented meaning: `man ditto` says `ditto src_directory
  dst_directory` "copies the contents of src_directory into
  dst_directory".
- `mv moved/ backups/` moved the folder itself; the slash made no
  difference for `mv` here.

## A copy that doesn't depend on the slash

If a script takes a folder path from a variable or an argument, don't let
a slash you can't see decide the result. In the cases I tried (no slash, one or two slashes, a symlink) this produces
`DEST/<folder name>`, merges on a second run instead of nesting, includes
dotfiles, follows a top-level symlink, and refuses a missing source:

```sh
# copydir.sh — usage: sh copydir.sh SRC DEST
# Copies the folder SRC to DEST/<name of SRC>, with or without a trailing slash.
set -eu
[ -d "$1" ] || { echo "copydir: not a directory: $1" >&2; exit 1; }
name=$(basename "$1")
mkdir -p "$2/$name"
cp -R "$1/." "$2/$name/"
```

`basename` drops trailing slashes (`project//` gives `project`), and
`"$1/."` copies the contents whether `$1` had a slash or not. In a new
empty folder with `copydir.sh` in it:

```console
$ mkdir -p project/sub && echo a > project/a.txt && echo b > project/sub/b.txt && echo x > project/.gitignore && ln -s project current
$ sh copydir.sh project backups && sh copydir.sh project/ backups && sh copydir.sh project// backups
$ find backups | sort
backups
backups/project
backups/project/.gitignore
backups/project/a.txt
backups/project/sub
backups/project/sub/b.txt
$ sh copydir.sh current/ backups; stat -f '%N: %HT%SY' backups/current
backups/current: Directory
$ sh copydir.sh missing backups; echo "exit $?"
copydir: not a directory: missing
exit 1
$ ls backups
current
project
```

Without the `[ -d "$1" ]` line, `mkdir -p` runs first, so a missing source
left an empty `backups/missing` behind before cp failed with
`cp: missing/.: No such file or directory`.

What this script doesn't do: it doesn't delete files that were removed from
the source since the last copy (nor does any `cp`), and symlinks *inside*
the tree are still copied as symlinks.

## Checklist

- `cp -R folder/ dest/` copies the **contents** of `folder` into `dest`
  (macOS `man cp`, `-R`). Exit 0, no warning. Check the variables that
  hold folder paths for a trailing slash.
- `cp -R folder dest` makes `dest` the copy if it doesn't exist, and
  `dest/folder` if it does: the second run nests. Either always copy into a
  folder of copies (`mkdir -p dest && cp -R folder dest/`) or always copy
  contents (`cp -R folder/. dest/`).
- `folder/*` skips dotfiles and fails on an empty folder (zsh and bash);
  `folder/.` doesn't.
- `cp -R link dest` copies the symlink, not the folder. Relative links
  dangle in another directory. Use `cp -RH` or `link/.`.
- `rsync` has the same slash rule; `ditto src dst` copies the contents with
  or without one; `mv folder/ dest/` moved the folder here.

*Tested on 2026-10-29 on macOS 26.5.2 (build 25F84, Apple silicon), APFS
`/tmp`, with `/bin/cp` and `/bin/mv` (file_cmds-479), `/usr/bin/rsync`
(openrsync, rsync-184, "rsync version 2.6.9 compatible"), `/usr/bin/ditto`,
`/usr/bin/stat`, in zsh 5.9 (`zsh -f`) and `/bin/sh` for the script; the empty-glob line also
in bash 3.2.57. GNU
cp, Samba rsync, Linux and other file systems were not tested.*
