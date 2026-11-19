---
title: "macOS `du -b`: \"invalid option -- b\" (exit 64) — and the size check that uploads anyway"
description: >-
  `du -sb dir` (or `du --apparent-size`) fails on a Mac with "du: invalid
  option -- b" and exit 64. In a script, `size=$(du -sb "$dir" | cut -f1)`
  is empty, `[ "$size" -gt "$limit" ]` prints "integer expression expected"
  and takes the else branch, so a 6 MB build passes a 5 MB limit and the
  script exits 0. macOS du cannot print bytes at all: `-A` is apparent size,
  `-B 1` is not bytes, `BLOCKSIZE=1` gives "minimum blocksize is 512". Fix:
  sum `stat -f %z` over `find -type f`, count hard links once, use pipefail.
tags: [macos, du, shell, bsd]
---

Getting the size of a folder in bytes, on a Mac:

```console
$ du -sb build; echo "exit $?"
du: invalid option -- b
usage: du [-Aclnx] [-H | -L | -P] [-g | -h | -k | -m] [-a | -s | -d depth] [-B blocksize] [-I mask] [-t threshold] [file ...]
exit 64
$ du -s --apparent-size build; echo "exit $?"
du: unrecognized option `--apparent-size'
usage: du [-Aclnx] [-H | -L | -P] [-g | -h | -k | -m] [-a | -s | -d depth] [-B blocksize] [-I mask] [-t threshold] [file ...]
exit 64
```

GNU `du -b` means "apparent size, in bytes" (that is how the GNU manual
describes it; I did not run GNU du for this post). The `du` that ships with
macOS (`/usr/bin/du`) has neither `-b` nor `--apparent-size` (the only long
option in its man page is `--si`). The error is loud on a terminal, but in a
script it disappears.

## The size check that passes a too-big build

A small pre-upload check: refuse anything over 5 MB.

```console
$ mkdir build
$ head -c 6000000 /dev/urandom > build/app.bin
$ printf 'hello\n' > build/index.html
$ cat upload-check.sh
#!/bin/sh
# refuse to upload a build bigger than 5 MB
dir=$1
limit=$((5 * 1024 * 1024))
size=$(du -sb "$dir" | cut -f1)
if [ "$size" -gt "$limit" ]; then
  echo "too big: $size bytes" >&2
  exit 1
fi
echo "ok: $size bytes, uploading"
$ sh upload-check.sh build; echo "exit $?"
du: invalid option -- b
usage: du [-Aclnx] [-H | -L | -P] [-g | -h | -k | -m] [-a | -s | -d depth] [-B blocksize] [-I mask] [-t threshold] [file ...]
upload-check.sh: line 6: [: : integer expression expected
ok:  bytes, uploading
exit 0
```

The build is 6,000,006 bytes, over the 5,242,880-byte limit, and the script
says "ok" and exits 0. Step by step:

- `du` writes its error and usage line to stderr and nothing to stdout, so
  `cut` gets no input and `size` is empty.
- The pipeline's status is `cut`'s, which is 0:

```console
$ du -sb build | cut -f1; echo "exit $?"
du: invalid option -- b
usage: du [-Aclnx] [-H | -L | -P] [-g | -h | -k | -m] [-a | -s | -d depth] [-B blocksize] [-I mask] [-t threshold] [file ...]
exit 0
```

- `[ "" -gt 5242880 ]` is an error ("integer expression expected"), and an
  error in `if` counts as false, so the script goes on to the upload line.

The output even has a hint, `ok:  bytes` with two spaces, but in a CI log
that is easy to miss.

## What macOS du can and cannot print

macOS `du` does have an apparent-size switch, `-A`. But every unit it offers is
a block: 512 bytes by default, or `-k`, `-m`, `-g`, and the rounded
human-readable `-h` (powers of 1024) and `--si` (powers of 1000). There is no
byte unit.

```console
$ du -s build
11728	build
$ du -sk build
5864	build
$ du -sAk build
5861	build
$ du -sAh build
5.7M	build
$ du -sA --si build
6.0M	build
```

The two obvious tries for "bytes" do not give bytes:

```console
$ du -sA -B 1 build
11720	build
$ BLOCKSIZE=1 du -sA build; echo "exit $?"
du: minimum blocksize is 512
11721	build
exit 0
```

- `-B 1` is not the output unit. The man page says `-B` "gives an estimate of
  how much space the examined file hierarchy would require on a filesystem
  with the given blocksize". The number printed is still in 512-byte blocks
  (11720 × 512 = 6,000,640).
- `BLOCKSIZE=1` is refused with a warning, du falls back to 512-byte blocks,
  and the exit status is still 0.

`-A` matters for sparse files, where disk usage and apparent size are far
apart. A 20 MB sparse file:

```console
$ mkdir disk
$ dd if=/dev/zero of=disk/vm.img bs=1 count=0 seek=20m 2>/dev/null
$ stat -f %z disk/vm.img
20971520
$ du -sk disk
0	disk
$ du -sAk disk
20481	disk
```

Without `-A`, the 20 MB file counts as 0 KB.

If a limit with a little slack is fine, `du -sAk` is enough: each file is
rounded up to whole KiB, so it reads slightly high (5861 KiB = 6,001,664 bytes
here, against 6,000,006 real bytes). For an exact byte count you have to add
up the file sizes yourself.

## Exact bytes: `find` + `stat -f %z`

```console
$ find build -type f -exec stat -f %z {} + | awk '{ s += $1 } END { print s + 0 }'
6000006
```

`stat -f %z` is the size in bytes (BSD `stat`; GNU's `stat -c %s` is
"illegal option -- c" here, see
[macOS `stat: illegal option -- c`](/2026/10/macos-stat-illegal-option-c-file-size-mtime-wc-padding/)). `-exec ... {} +` passes many files per `stat`
call and is fine with spaces in names. This counts regular files only, not
directory entries or symlinks.

Two things this simple version gets wrong.

**Hard links are counted twice.** `du` counts a file with several links once;
the sum above counts each name:

```console
$ ln build/app.bin build/app-copy.bin
$ du -sAk build
5861	build
$ find build -type f -exec stat -f %z {} + | awk '{ s += $1 } END { print s + 0 }'
12000006
$ find build -type f -exec stat -f '%i %z' {} + | awk '!seen[$1]++ { s += $2 } END { print s + 0 }'
6000006
$ rm build/app-copy.bin
```

`%i` is the inode number; the awk keeps the first name of each inode. (Inode
numbers are unique per filesystem; if something may be mounted inside the
folder, `find -x` stays on one filesystem. I did not test a mount here.)

**Errors still turn into a number.** A missing folder gives `0`, exit 0:

```console
$ find nosuch -type f -exec stat -f %z {} + | awk '{ s += $1 } END { print s + 0 }'; echo "exit $?"
find: nosuch: No such file or directory
0
exit 0
```

An unreadable subfolder is worse: its files are skipped and you get a total
that looks right but is too small, still with exit 0:

```console
$ mkdir small
$ printf 'hello\n' > small/index.html
$ printf 'x\n' > 'small/a b.txt'
$ mkdir small/private
$ printf 'secret\n' > small/private/key
$ chmod 000 small/private
$ find small -type f -exec stat -f %z {} + | awk '{ s += $1 } END { print s + 0 }'; echo "exit $?"
find: small/private: Permission denied
8
exit 0
$ chmod 755 small/private
$ find small -type f -exec stat -f %z {} + | awk '{ s += $1 } END { print s + 0 }'
15
```

`set -o pipefail` makes `find`'s failure the pipeline's status.

## The fixed check

```console
$ cat upload-check.sh
#!/bin/sh
# refuse to upload a build bigger than 5 MB
set -u -o pipefail
dir=${1:?usage: upload-check.sh DIR}
limit=$((5 * 1024 * 1024))
size=$(find "$dir" -type f -exec stat -f '%i %z' {} + |
  awk '!seen[$1]++ { s += $2 } END { print s + 0 }') || {
  echo "could not measure $dir" >&2
  exit 2
}
if [ "$size" -gt "$limit" ]; then
  echo "too big: $size bytes" >&2
  exit 1
fi
echo "ok: $size bytes, uploading"
$ sh upload-check.sh build; echo "exit $?"
too big: 6000006 bytes
exit 1
$ mv small/private/key private-key
$ sh upload-check.sh small; echo "exit $?"
ok: 8 bytes, uploading
exit 0
$ sh upload-check.sh nosuch; echo "exit $?"
find: nosuch: No such file or directory
could not measure nosuch
exit 2
$ mv private-key small/private/key
$ chmod 000 small/private
$ sh upload-check.sh small; echo "exit $?"
find: small/private: Permission denied
could not measure small
exit 2
$ chmod 755 small/private
$ sh upload-check.sh; echo "exit $?"
upload-check.sh: line 4: 1: usage: upload-check.sh DIR
exit 1
```

On my Mac `/bin/sh` runs bash 3.2.57 (the default; `/private/var/select/sh`
points to `/bin/bash`), which has `pipefail`. I only ran it with that `sh`;
if your script may run under another `sh`, use `#!/bin/bash`.

## Checklist

- macOS `du` has no `-b` and no `--apparent-size` (the only long option
  its man page lists is `--si`): `-b` is "invalid option -- b", `--apparent-size` is "unrecognized
  option", both exit 64.
- The error goes to stderr. `$(du -sb dir | cut -f1)` is empty and the
  pipeline exits 0; a following `[ "$size" -gt ... ]` errors and the `if`
  takes the else branch.
- `du -A` is apparent size, but there is no byte unit (512-byte blocks by
  default, `-k`, `-m`, `-g`, `-h`, `--si`). `-B 1` changes rounding, not the unit;
  `BLOCKSIZE=1` prints "minimum blocksize is 512" and exits 0.
- Without `-A`, sparse files count only the blocks they use (a 20 MB sparse
  file is 0 KB).
- Exact bytes: `find DIR -type f -exec stat -f '%i %z' {} + | awk '!seen[$1]++ { s += $2 } END { print s + 0 }'`.
- That pipeline prints `0` for a missing folder and skips unreadable ones; run
  it with `set -o pipefail` and check the status.

Related: [macOS `stat: illegal option -- c`](/2026/10/macos-stat-illegal-option-c-file-size-mtime-wc-padding/)
and [macOS `cp --parents` / `du --max-depth`](/2026/10/macos-cp-parents-illegal-option-pax-rsync-r/).

*Tested on 2026-11-19 on macOS 26.5.2 (Apple silicon, APFS) with `/usr/bin/du`
and `/usr/bin/stat` (file_cmds-479), `/usr/bin/find` (shell_cmds-329), `/usr/bin/awk`, zsh 5.9 and `/bin/sh`
(bash 3.2.57), in `env -i` with `PATH=/usr/bin:/bin:/usr/sbin:/sbin`.
Not tested: GNU coreutils `du`/`stat`, Linux, other filesystems (HFS+,
exFAT, network shares), older macOS versions.*
