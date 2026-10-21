---
title: "macOS `head: illegal line count -- -1`: dropping the last lines with `sed '$d'` or awk, and why `tail -r` glues a line without a trailing newline"
description: >-
  `head -n -1 file` fails on macOS with "head: illegal line count -- -1"
  (and `head -c -1` with "illegal byte count") because /usr/bin/head is BSD
  head. In a pipeline the error goes to stderr and the rest of the pipeline
  sees no input. To drop the last line use `sed '$d'`; for the last N lines,
  a small awk ring buffer. The `tail -r | tail -n +2 | tail -r` trick lost
  two lines on a file without a final newline in my tests, because `tail -r`
  joined the last two lines. Reproduced on macOS 26.
tags: [macos, shell, head, bsd]
---

A script that drops the last line of a file stops on a Mac:

```console
$ mkdir demo && cd demo
$ printf '1\n2\n3\n4\n5\n' > n.txt
$ head -n -1 n.txt; echo "exit $?"
head: illegal line count -- -1
exit 1
```

A negative count ("everything except the last N") is a GNU head feature.
macOS ships BSD head, which only takes a positive count. The byte form fails
the same way:

```console
$ head -c -1 n.txt; echo "exit $?"
head: illegal byte count -- -1
exit 1
```

There's no `--version`; `what` shows the build:

```console
$ head --version; echo "exit $?"
head: unrecognized option `--version'
usage: head [-n lines | -c bytes] [file ...]
exit 1
$ what /usr/bin/head /usr/bin/tail /usr/bin/sed /usr/bin/awk
/usr/bin/head:
	PROGRAM:head  PROJECT:text_cmds-199
	PROGRAM:head  PROJECT:text_cmds-199
/usr/bin/tail:
	PROGRAM:tail  PROJECT:text_cmds-199
	PROGRAM:tail  PROJECT:text_cmds-199
/usr/bin/sed:
	PROGRAM:sed  PROJECT:text_cmds-199
	PROGRAM:sed  PROJECT:text_cmds-199
/usr/bin/awk:
	PROGRAM:awk  PROJECT:awk-40
	PROGRAM:awk  PROJECT:awk-40
```

## In a pipeline it looks like empty input

When `head -n -1` sits in the middle of a pipeline, the next command gets
nothing and carries on. Without `pipefail`, the pipeline's exit status is the
last command's, so a script sees success:

```console
$ seq 5 | head -n -1 | wc -l; echo "exit $?"
head: illegal line count -- -1
       0
exit 0
```

If the stderr line scrolls past in a log, what's left is a count of 0 and
exit 0.

## Drop the last line: `sed '$d'`

`$` is the last line and `d` deletes it:

```console
$ sed '$d' n.txt
1
2
3
4
```

## Drop the last N lines: awk ring buffer

Hold the last `n` lines and print a line only once `n` newer ones have
arrived:

```console
$ awk -v n=2 'NR>n{print buf[NR%n]} {buf[NR%n]=$0}' n.txt
1
2
3
```

`n` has to be 1 or more. With `n=0` the `%` divides by zero:

```console
$ awk -v n=0 'NR>n{print buf[NR%n]} {buf[NR%n]=$0}' n.txt; echo "exit $?"
awk: division by zero in mod
 input record number 1, file n.txt
 source line number 1
exit 2
```

If `n` is larger than the file, nothing is printed:

```console
$ awk -v n=10 'NR>n{print buf[NR%n]} {buf[NR%n]=$0}' n.txt | wc -l
       0
```

## The `tail -r` trick and a file without a final newline

macOS has no `tac`, but `tail -r` prints lines in reverse, so another common
answer is "reverse, skip the first N, reverse back". On a normal file it
works:

```console
$ tail -r n.txt | tail -n +3 | tail -r
1
2
3
```

Now take a file whose last line has no newline, for example one
written by `printf` without a final `\n`:

```console
$ printf '1\n2\n3' > nonl.txt
$ tail -r nonl.txt | od -c
0000000    3   2  \n   1  \n
0000005
```

`tail -r` put the unterminated `3` first, but didn't add a newline after it,
so it ran straight into `2`. Dropping "one line" from that drops two:

```console
$ tail -r nonl.txt | tail -n +2 | tail -r
1
$ sed '$d' nonl.txt
1
2
```

`sed '$d'` and the awk version both treat the unterminated `3` as the last
line and drop only that:

```console
$ awk -v n=1 'NR>n{print buf[NR%n]} {buf[NR%n]=$0}' nonl.txt
1
2
```

## Instead of `head -c -1`

If what you wanted from `head -c -1` was to strip a trailing newline, Perl (in
`/usr/bin` on this Mac) can remove the newline at the end of the input only:

```console
$ printf 'abc\n' | perl -pe 'chomp if eof' | od -c
0000000    a   b   c
0000003
$ printf 'abc\n\n' | perl -pe 'chomp if eof' | od -c
0000000    a   b   c  \n
0000004
```

It removes exactly one final newline. I only checked it for that, not as a
general "drop the last N bytes".

## Checklist

- `head: illegal line count -- -1` / `illegal byte count -- -1`: BSD head on
  macOS takes no negative counts.
- In a pipeline the next command just gets no input; without `pipefail` the
  pipeline's exit status is the last command's (0 here).
- Last line: `sed '$d'`.
- Last N lines (N ≥ 1): `awk -v n=N 'NR>n{print buf[NR%n]} {buf[NR%n]=$0}'`.
- `tail -r | tail -n +2 | tail -r` lost two lines on a file without a final
  newline in my tests; `sed '$d'` and the awk version didn't.
- Strip one final newline: `perl -pe 'chomp if eof'`.

*Tested on macOS 26.5.2 (Apple silicon) with `/usr/bin/head`, `/usr/bin/tail`,
`/usr/bin/sed` (`text_cmds-199`), `/usr/bin/awk` (`awk-40`, "awk version
20200816"), perl 5.34.1, in zsh 5.9 (arm64-apple-darwin25.0). GNU coreutils
and Linux were not tested.*
