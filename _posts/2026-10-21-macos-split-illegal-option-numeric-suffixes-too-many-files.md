---
title: "macOS `split: illegal option -- -` for `--numeric-suffixes`: use `-d`, and why `split -d` stops at 100 files with \"too many files\""
description: >-
  `split --numeric-suffixes` and `--additional-suffix` fail on macOS with
  "split: illegal option -- -" (exit 64) because /usr/bin/split is BSD split.
  `split -d` gives numeric suffixes, but in my tests the suffix stayed two
  characters: the 101st piece failed with "split: too many files" (exit 65),
  after 100 files had been written and the rest of the input was missing.
  Set `-a` from the number of pieces, and add an extension with a rename loop.
  Reproduced on macOS 26.
tags: [macos, shell, split, bsd]
---

A script that splits a file into numbered pieces stops on a Mac:

```console
$ mkdir demo && cd demo
$ seq 10 > in.txt
$ split --numeric-suffixes=1 -l 4 in.txt part_; echo "exit $?"
split: illegal option -- -
usage: split [-cd] [-l line_count] [-a suffix_length] [file [prefix]]
       split [-cd] -b byte_count[K|k|M|m|G|g] [-a suffix_length] [file [prefix]]
       split [-cd] -n chunk_count [-a suffix_length] [file [prefix]]
       split [-cd] -p pattern [-a suffix_length] [file [prefix]]
exit 64
$ split --additional-suffix=.txt -l 4 in.txt part_; echo "exit $?"
split: illegal option -- -
usage: split [-cd] [-l line_count] [-a suffix_length] [file [prefix]]
       split [-cd] -b byte_count[K|k|M|m|G|g] [-a suffix_length] [file [prefix]]
       split [-cd] -n chunk_count [-a suffix_length] [file [prefix]]
       split [-cd] -p pattern [-a suffix_length] [file [prefix]]
exit 64
```

Long options like these are GNU split's (from GNU's documentation; I didn't
run GNU split here). macOS ships BSD split, and its usage text above lists
no long options. `what` shows the build:

```console
$ what /usr/bin/split
/usr/bin/split:
	PROGRAM:split  PROJECT:text_cmds-199
	PROGRAM:split  PROJECT:text_cmds-199
```

## `-d`: numeric suffixes, starting at 00

The usage line lists `-d`. It gives numbers instead of letters:

```console
$ split -d -l 4 in.txt part_; echo "exit $?"
exit 0
$ ls part_*
part_00
part_01
part_02
```

The numbers start at `00`. `--numeric-suffixes=1` (start at 01) has no BSD
flag in the usage line above.

## The 101st piece: "too many files"

The suffix has two characters unless you say otherwise. From the man page:

```console
$ man split | col -b | grep -B1 'letters are used'
     suffix_length characters in the range “a-z”.  If -a is not specified, two
     letters are used as the initial suffix.
```

With two digits there are 100 names, `00` to `99`. Split 1005 lines into
10-line pieces (101 pieces needed):

```console
$ seq 1005 > big.txt
$ split -d -l 10 big.txt n_; echo "exit $?"
split: too many files
exit 65
$ ls n_* | wc -l
     100
$ cat n_* | wc -l
    1000
$ tail -1 n_99
1000
$ cat n_* | cmp - big.txt; echo "exit $?"
cmp: EOF on stdin
exit 1
```

The suffix didn't grow to three digits. split wrote 100 files, then stopped
with exit 65, and lines 1001–1005 are in none of them. A script that ignores
the exit status ends up with a quietly short copy.

Letters ran out the same way, after `zz` (26 × 26 = 676 files):

```console
$ split -l 1 big.txt L_; echo "exit $?"
split: too many files
exit 65
$ ls L_* | wc -l
     676
```

## Fix: set `-a` from the number of pieces

Count the pieces, and use as many digits as that number has:

```console
$ n=10; lines=$(wc -l < big.txt); pieces=$(( (lines + n - 1) / n )); echo "$pieces pieces, -a ${#pieces}"
101 pieces, -a 3
$ split -d -a "${#pieces}" -l "$n" big.txt k_; echo "exit $?"
exit 0
$ ls k_* | wc -l
     101
$ cat k_* | cmp - big.txt && echo "same as input"
same as input
```

`${#pieces}` is the length of the number (101 → 3). When the piece count is
exactly a power of ten, this gives one more digit than needed, which only
makes the names longer:

```console
$ seq 1000 > even.txt
$ lines=$(wc -l < even.txt); pieces=$(( (lines + n - 1) / n )); echo "$pieces pieces, -a ${#pieces}"
100 pieces, -a 3
```

I only tested this with `-l`.

## Instead of `--additional-suffix`: rename after

```console
$ for f in part_??; do mv -- "$f" "$f.txt"; done
$ ls part_*
part_00.txt
part_01.txt
part_02.txt
```

`part_??` matches exactly two characters after `part_`. With `-a 3`, use
`part_???`.

## Checklist

- `split: illegal option -- -` (exit 64): `/usr/bin/split` on macOS is BSD
  split; its usage text lists no long options.
- Numeric suffixes: `-d`. They start at `00`.
- Default suffix length is 2. In my tests the 101st `-d` piece (or the
  677th letter piece) failed with "split: too many files", exit 65, and the
  rest of the input was not written.
- Set the length: `-a "${#pieces}"` with
  `pieces=$(( (lines + n - 1) / n ))`.
- Check split's exit status, or compare `cat prefix* | cmp - input`.
- Extension: rename afterwards, `for f in part_??; do mv -- "$f" "$f.txt"; done`.

*Tested on macOS 26.5.2 (Apple silicon) with `/usr/bin/split`
(`PROGRAM:split  PROJECT:text_cmds-199`) in zsh 5.9 (arm64-apple-darwin25.0).
GNU coreutils, bash, and Linux were not tested; `split -b` and `-n` were
not tested.*
