---
title: "macOS `mv: illegal option -- t` — `xargs mv -t archive/` moves nothing and the script still says it did"
description: >-
  macOS mv, cp, ln and install have no -t (--target-directory). The Linux
  idiom `find logs -mtime +7 -print0 | xargs -0 -r mv -t archive/` prints
  "mv: illegal option -- t" and the usage text, moves no file, and a script
  that echoes "archived" after it exits 0. mv exits 64, xargs turns that
  into 1, the pipeline status is xargs's, and the next line hides it all.
  `find -exec mv {} archive/ +` is rejected by macOS find (`find: -exec: no
  terminating ";" or "+"`) because {} must come right before +. Working
  replacements: `-exec sh -c 'mv "$@" archive/' sh {} +`, or macOS-only
  `xargs -0 -J % mv % archive/`. Keep the trailing slash: with one file and
  a missing folder, `mv file archive` renames the file to "archive".
tags: [macos, mv, xargs, find, shell, bash, bsd]
---

A log-rotation script copied from a Linux box, run on a Mac:

```console
$ ./archive-linux.sh; echo "exit $?"
mv: illegal option -- t
usage: mv [-f | -i | -n] [-hv] source target
       mv [-f | -i | -n] [-v] source ... directory
archived old logs
exit 0
$ ls archive
$ ls logs
app1.log
app2.log
app3.log
my app.log
```

The script said "archived old logs" and exited 0. `archive/` is empty;
every log is still in `logs/`. From a scheduled job, where stderr goes to a
log nobody reads, it looks like a job that works.

## Reproduce

Everything runs in a scratch folder. Three logs are made 20 days old, one is
new, and one name has a space in it:

```console
$ T=$(mktemp -d)
$ cd "$T"
$ mkdir logs archive
$ for f in app1 app2 app3 'my app'; do echo x > "logs/$f.log"; done
$ touch -t "$(date -v-20d +%Y%m%d%H%M)" logs/app1.log logs/app2.log 'logs/my app.log'
$ find logs -name '*.log' -mtime +7
logs/my app.log
logs/app2.log
logs/app1.log
```

(`find` prints in directory order; yours may differ.)

Save this as `archive-linux.sh` — the usual GNU way to move a list of files
into a folder:

```bash
#!/bin/bash
find logs -name '*.log' -mtime +7 -print0 | xargs -0 -r mv -t archive/
echo "archived old logs"
```

```console
$ chmod +x archive-linux.sh
$ ./archive-linux.sh; echo "exit $?"
mv: illegal option -- t
usage: mv [-f | -i | -n] [-hv] source target
       mv [-f | -i | -n] [-v] source ... directory
archived old logs
exit 0
```

Where the error went:

- `mv -t` is an illegal option on macOS; `mv` exits 64 (`EX_USAGE`) and
  moves nothing:

  ```console
  $ mv -t archive logs/app1.log; echo "exit $?"
  mv: illegal option -- t
  usage: mv [-f | -i | -n] [-hv] source target
         mv [-f | -i | -n] [-v] source ... directory
  exit 64
  ```

- `xargs` turns that into its own exit 1 (its man page: "If any other error
  occurs, xargs exits with a value of 1"). `find` itself was fine:

  ```console
  $ bash -c 'find logs -name "*.log" -mtime +7 -print0 | xargs -0 -r mv -t archive/; echo "${PIPESTATUS[*]}"'
  mv: illegal option -- t
  usage: mv [-f | -i | -n] [-hv] source target
         mv [-f | -i | -n] [-v] source ... directory
  0 1
  ```

- Nothing checks that 1. The script goes on to `echo`, and a script's exit
  status is its last command's.

`-r` is not the problem: macOS `xargs` accepts it, and with empty input it
runs nothing with or without it. If `mv` had run here, its usage text would
show:

```console
$ printf '' | xargs -0 -r mv archive/; echo "exit $?"
exit 0
$ printf '' | xargs -0 mv archive/; echo "exit $?"
exit 0
```

The same `-t` is missing from `cp`, `ln` and `install`:

```console
$ cp -t archive logs/app1.log; echo "exit $?"
cp: illegal option -- t
usage: cp [-R [-H | -L | -P]] [-fi | -n] [-aclpSsvXx] source_file target_file
       cp [-R [-H | -L | -P]] [-fi | -n] [-aclpSsvXx] source_file ... target_directory
exit 64
$ ln -t archive logs/app1.log; echo "exit $?"
ln: illegal option -- t
usage: ln [-s [-F] | -L | -P] [-f | -i] [-hnv] source_file [target_file]
       ln [-s [-F] | -L | -P] [-f | -i] [-hnv] source_file ... target_dir
exit 1
$ install -t archive logs/app1.log 2>&1 | head -1
install: illegal option -- t
$ install -t archive logs/app1.log 2>/dev/null; echo "exit $?"
exit 64
```

(`install` prints a long usage text after that first line.)

## Why scripts use `-t` at all

`xargs` (and `find -exec ... {} +`) put the file names at the **end** of the
command. `mv` wants the destination last. GNU `mv -t DIR` moves the
destination to the front so the names can go at the end. Without it you need
another way to get the destination after the names.

## The obvious rewrite fails too: `-exec mv {} archive/ +`

```console
$ find logs -name '*.log' -mtime +7 -exec mv {} archive/ +; echo "exit $?"
find: -exec: no terminating ";" or "+"
exit 1
```

macOS `find` only accepts the batch form as `-exec utility [argument ...] {} +`
— `{}` has to be the last thing before `+`. (GNU find rejects it too, per its
manual; not tested here.) `-exec mv {} archive/ \;` works, but runs one `mv`
per file.

## Fix 1: put the destination after `"$@"` in a tiny `sh -c`

```console
$ find logs -name '*.log' -mtime +7 -exec sh -c 'mv "$@" archive/' sh {} +; echo "exit $?"
exit 0
$ ls archive
app1.log
app2.log
my app.log
$ ls logs
app3.log
```

`find` passes the names as `$1 ...`; the `sh` after the script is `$0`.
`"$@"` keeps the name with a space whole. It uses only what POSIX specifies
for `find -exec ... {} +` and `sh -c`, so it should run the same on Linux;
I have not tested it there.

If `mv` fails, `find` returns non-zero (the man page: "if at least one
invocation of utility returns a non-zero exit status, find will return a
non-zero exit status"). Put the files back and send them to a folder that
does not exist:

```console
$ mv archive/* logs/
$ find logs -name '*.log' -mtime +7 -exec sh -c 'mv "$@" nodir/' sh {} +; echo "exit $?"
mv: nodir/ is not a directory
exit 1
```

(`mv` within one file system keeps the modification time, so the moved-back
files are still 20 days old.)

## Fix 2 (macOS/BSD only): `xargs -J`

macOS `xargs -J %` puts the input where `%` is, instead of at the end:

```console
$ find logs -name '*.log' -mtime +7 -print0 | xargs -0 -r -J % mv % archive/; echo "exit $?"
exit 0
$ ls archive
app1.log
app2.log
my app.log
$ mv archive/* logs/
```

From the man page: `%` "must show up as a distinct argument", and "only the
first occurrence of the replstr will be replaced". GNU `xargs` has no `-J`
(per its manual; not tested here), so this line is not portable the other
way. `xargs -I {} mv {} archive/` is in POSIX and works on this Mac, one
`mv` per file (not tested with GNU `xargs`).

## Keep the trailing slash on the destination

The slash in `archive/` is not decoration. When the list has only one file
and the folder is missing (a typo, a folder not made yet), `mv file archive`
is a rename:

```console
$ find logs -name 'app3.log' -exec sh -c 'mv "$@" old' sh {} +; echo "exit $?"
exit 0
$ ls
archive
archive-linux.sh
logs
old
$ mv old logs/app3.log
$ find logs -name 'app3.log' -exec sh -c 'mv "$@" old/' sh {} +; echo "exit $?"
mv: rename logs/app3.log to old/: No such file or directory
exit 1
```

Without the slash, `logs/app3.log` became a file named `old`, exit 0. With
it, `mv` refuses.

## A script that fails when the move fails

Save this as `archive.sh`:

```bash
#!/bin/bash
# Move *.log files older than 7 days from logs/ into archive/.
# macOS mv has no -t, so put the destination after "$@" in sh -c.
src=logs
dest=archive

mkdir -p "$dest" || exit 1
find "$src" -name '*.log' -mtime +7 \
  -exec sh -c 'd=$1; shift; mv "$@" "$d/"' sh "$dest" {} + || {
  echo "archiving failed" >&2
  exit 1
}
echo "archived old logs"
```

The destination goes in as `$1` so it stays a variable (no quoting games
inside the single-quoted script); `shift` leaves only the file names in `"$@"`.

```console
$ chmod +x archive.sh
$ ./archive.sh; echo "exit $?"
archived old logs
exit 0
$ ls archive
app1.log
app2.log
my app.log
$ ls logs
app3.log
```

Make the move fail — a folder you can't write to:

```console
$ mv archive/* logs/
$ chmod 555 archive
$ ./archive.sh; echo "exit $?"
mv: rename logs/my app.log to archive/my app.log: Permission denied
mv: rename logs/app2.log to archive/app2.log: Permission denied
mv: rename logs/app1.log to archive/app1.log: Permission denied
archiving failed
exit 1
$ chmod 755 archive
```

Now the scheduler sees exit 1 instead of a quiet 0. (The order of the
`Permission denied` lines follows `find`'s order.)

Note `-mtime +7` on this Mac (default `COMMAND_MODE`) means 8 days or more,
not "older than 7 days" —
see [the `find -mtime +7` post](/2026/11/macos-find-mtime-plus-7-not-deleting-7-day-old-files-rounded-up/).

Clean up:

```console
$ cd /
$ rm -r "${T:?}"
```

## Checklist

- macOS `mv`, `cp`, `ln` and `install` have no `-t`: "illegal option -- t",
  nothing moved, exit 64 (`ln`: 1).
- Behind `xargs`, that becomes exit 1 from `xargs`, which a script without
  a check (or the next `echo`) hides.
- `find ... -exec mv {} archive/ +` is a syntax error on macOS: `{}` must be
  right before `+`.
- POSIX-only fix (tested on macOS, not on Linux):
  `find ... -exec sh -c 'mv "$@" archive/' sh {} +`. macOS/BSD only:
  `xargs -0 -J % mv % archive/`. One `mv` per file:
  `-exec mv {} archive/ \;` or `xargs -I {} mv {} archive/`.
- Write the destination as `archive/`, with the slash: without it one file
  plus a missing folder is a silent rename.
- Check the status of the line that moves (`|| exit 1`), not just the end
  of the script.

Related: [macOS `xargs: invalid option -- d`](/2026/10/macos-xargs-invalid-option-d-i-command-line-cannot-be-assembled/)
and [macOS `cp --parents: illegal option`](/2026/10/macos-cp-parents-illegal-option-pax-rsync-r/).

*Tested on 2026-11-27 on macOS 26.5.2 (Apple silicon, APFS) with `/bin/mv`
(file_cmds-479), `/usr/bin/find` and `/usr/bin/xargs` (shell_cmds-329),
`/bin/bash` 3.2.57, zsh 5.9, in `env -i` with
`PATH=/usr/bin:/bin:/usr/sbin:/sbin`.
Not tested: GNU coreutils/findutils (Linux or Homebrew), names with
newlines, network or non-APFS file systems, older macOS versions.*
