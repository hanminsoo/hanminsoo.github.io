---
title: "`ln -sf` doesn't replace a symlink to a directory: it makes a link *inside* it (exit 0, \"Too many levels of symbolic links\") — use `ln -sfn` / `mv -fh`"
description: >-
  On macOS, `ln -sf v2 current` where `current` is a symlink to the directory
  v1 exits 0 but leaves `current -> v1` and creates `v1/v2 -> v2`, a link to
  itself ("Too many levels of symbolic links"). With an absolute target the
  stray link even works, so nothing looks wrong. `ln -sfn` (or `-sfh`) replaces
  the link itself, but still puts the link inside if `current` is a real
  directory, and `-F` silently removes an empty one. `ln -sfn` also unlinks
  before it creates: readers saw "No such file or directory" during a swap.
  `ln -s v2 next && mv -fh next current` didn't show that; plain `mv -f` moves
  the link into the directory instead.
tags: [macos, shell, ln, symlink, mv, deploy]
---

A deploy script switches a `current` symlink from one release folder to the
next. It runs without errors, and `current` still points at the old one:

```console
$ mkdir v1 v2 && echo one > v1/app.txt && echo two > v2/app.txt
$ ln -s v1 current
$ ln -sf v2 current; echo "exit $?"
exit 0
$ readlink current
v1
```

`-f` means "if the target exists, remove it first", so this looks like it
should work. It doesn't, because `current` is a symlink **to a directory**,
and `ln` follows it. On this Mac, the new link went into the old release:

```console
$ ls v1
app.txt
v2
$ stat -f '%N -> %Y' v1/v2
v1/v2 -> v2
$ cat v1/v2/app.txt; echo "exit $?"
cat: v1/v2/app.txt: Too many levels of symbolic links
exit 1
$ ln -sf v2 current; echo "exit $?"
exit 0
```

`ln` read `ln -sf v2 current` as "make a link named `v2` in the directory
`current`", the same as `ln -s v2 somedir/`. The link text is the relative
`v2`, so `v1/v2` resolves to `v1/v2`: a link to itself. Running it again
replaces that stray link (`-f`) and exits 0 again. The deploy never fails;
it just never switches.

## With an absolute path nothing looks wrong

With an absolute target (common in scripts, `"$PWD/v2"`, `"$RELEASES/$id"`), the stray
link isn't broken. It works, so there's no error anywhere:

```console
$ rm v1/v2
$ ln -sf "$PWD/v2" current; echo "exit $?"
exit 0
$ readlink current
v1
$ cat v1/v2/app.txt
two
```

`current` still serves `v1`, and `v1` now has a working `v2` folder-link
inside it. After a few deploys, each old release has a link to the next
one, and `current` hasn't moved.

## Fix: `-n` (or `-h`) — don't follow the target

BSD `ln` has `-h`: "If the target_file or target_dir is a symbolic link, do
not follow it." `-n` is listed as "Same as -h, for compatibility with other
ln implementations." With either, `-f` removes the link `current` itself:

```console
$ rm v1/v2
$ ln -sfn v2 current; echo "exit $?"
exit 0
$ readlink current; cat current/app.txt
v2
two
$ ls v1 v2
v1:
app.txt

v2:
app.txt
$ ln -sfh v1 current; readlink current
v1
```

No stray links this time. If you've already run `ln -sf` deploys, look for
links inside the release folders (`find v1 v2 -type l` here) and remove
them.

## `-n` only helps if `current` is a symlink

`-n` means "don't follow a *symlink*". If `current` is a real directory (an
earlier step copied files there, or it was never a link), `ln -sfn` still
puts the link inside it and exits 0:

```console
$ rm current && mkdir current
$ ln -sfn v2 current; echo "exit $?"
exit 0
$ ls current
v2
```

BSD `ln` also has `-F`: if the target is a directory, remove it so the link
can take its place. On a non-empty directory it fails. On an **empty** one it
removed the directory without asking:

```console
$ ln -sFn v2 current; echo "exit $?"
ln: current: Directory not empty
exit 1
$ rm current/v2
$ ln -sFn v2 current; echo "exit $?"
exit 0
$ readlink current
v2
```

I wouldn't put `-F` in a deploy script. If `current` is a directory when you
expect a link, stop and look; `[ -L current ] || { echo "current is not a symlink" >&2; exit 1; }` before the
switch does that.

## `ln -sfn` leaves a moment with no `current`

`-f` is "unlink it so that the link may occur": first remove, then create.
Between the two, `current` doesn't exist. To see that, this script swaps
`current` between `v1` and `v2` 3000 times while the foreground loop reads
`current/app.txt` as fast as it can:

```sh
# race.sh — usage: zsh -f race.sh lnsfn|mvh
mode=$1
swap() {
  i=0
  while [ $i -lt 3000 ]; do
    t=v1; [ $((i % 2)) -eq 0 ] && t=v2
    if [ "$mode" = lnsfn ]; then
      ln -sfn "$t" current
    else
      ln -s "$t" current.tmp && mv -fh current.tmp current
    fi
    i=$((i + 1))
  done
}
swap &
S=$!
miss=0 reads=0
while kill -0 $S 2>/dev/null; do
  cat current/app.txt 2>>err.$mode >/dev/null || miss=$((miss + 1))
  reads=$((reads + 1))
done
wait $S
echo "$mode: $reads reads, $miss failed"
readlink current
```

Saved as `race.sh` in a new empty folder (not the one above):

```console
$ mkdir v1 v2 && echo one > v1/app.txt && echo two > v2/app.txt && ln -s v1 current
$ zsh -f race.sh lnsfn
lnsfn: 3184 reads, 58 failed
v1
$ sort err.lnsfn | uniq -c
   4 cat: current/app.txt: Invalid argument
  54 cat: current/app.txt: No such file or directory
```

The counts change from run to run. In seven runs from a fresh `err` file,
42 to 64 reads got `No such file or directory` (and 0 to 4 got
`Invalid argument`, below). That's the gap
between unlink and symlink. Anything reading through `current` at that
moment (a web server, a launchd job) gets the same error; I only tested
`cat`.

The usual fix is to build the new link under another name and rename it
over the old one, so the name `current` is replaced rather than removed and
recreated. With `mv`
there's the same trap as with `ln`: plain `mv -f next current` follows
`current` into the directory:

```console
$ ln -s v1 next && mv -f next current; echo "exit $?"
exit 0
$ readlink current; stat -f '%N -> %Y' v2/next
v2
v2/next -> v1
$ rm v2/next
$ ln -s v1 next && mv -fh next current; echo "exit $?"
exit 0
$ readlink current; ls
v1
current
v1
v2
```

BSD `mv -h`: "If the target operand is a symbolic link to a directory, do
not follow it." With it, `next` replaced `current`. In the race (same
race folder as before):

```console
$ zsh -f race.sh mvh
mvh: 6003 reads, 6 failed
v1
$ sort err.mvh | uniq -c
   6 cat: current/app.txt: Invalid argument
```

No `No such file or directory` in seven `mvh` runs. But not zero failures
(0 to 10 per run):
a few reads got `Invalid argument` (EINVAL), with `mv -fh` and with
`ln -sfn`. I haven't found out why reading through a symlink that's
being renamed over returns EINVAL on this APFS volume, so I can only say
what I saw: the `mv -fh` swap removed the missing-file window, and a reader
that must never fail should still retry once.

## Checklist

- `ln -sf NEW current` with `current -> some_dir` exits 0, leaves `current`
  unchanged and creates `some_dir/NEW`. Relative target: a self-loop
  ("Too many levels of symbolic links"). Absolute target: a working stray
  link, so no error at all.
- Use `ln -sfn` (BSD also `-sfh`) to replace the link itself.
- `-n` doesn't help if `current` is a real directory: the link still goes
  inside. Check `[ -L current ]` first. BSD `-F` silently removes an empty
  directory; I wouldn't use it.
- `ln -sfn` is unlink + symlink: readers saw "No such file or directory"
  during swaps here. `ln -s NEW next && mv -fh next current` didn't show that.
- `mv -f next current` without `-h` moves `next` into the directory
  `current` points to.
- Already deployed with `ln -sf`? `find <release dirs> -type l` and remove
  the stray links.

*Tested on 2026-10-26 on macOS 26.5.2 (build 25F84, Apple silicon), APFS
data volume (`/tmp`), with `/bin/ln` and `/bin/mv` (file_cmds-479),
`/usr/bin/stat`, `/usr/bin/python3` 3.8.2, in zsh 5.9 (`zsh -f`). Race
counts vary per run. GNU coreutils `ln`/`mv` (`-T`), Linux, other file systems and
the cause of the EINVAL reads were not tested.*
