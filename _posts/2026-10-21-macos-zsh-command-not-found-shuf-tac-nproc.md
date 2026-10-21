---
title: "macOS `zsh: command not found: shuf` / `tac` / `nproc`: `sort -R` groups duplicate lines, and an empty `$(nproc)` makes `make -j` unlimited"
description: >-
  macOS doesn't ship `shuf`, `tac` or `nproc`, so scripts stop with
  "zsh: command not found: shuf" (exit 127). Replacements that ship with
  macOS: `sysctl -n hw.logicalcpu` or `getconf _NPROCESSORS_ONLN` for
  nproc, `tail -r` for tac, `sort -R` or Perl's List::Util shuffle for shuf.
  The traps: `make -j"$(nproc)"` becomes `make -j` with no limit, `sort -R`
  keeps identical lines together so `sort -R | head -n 1` picks by distinct
  line, and awk's `srand()` repeats within the same second. Reproduced on
  macOS 26.
tags: [macos, shell, shuf, tac, nproc, bsd]
---

A script copied from a Linux box stops on a Mac at the first of these:

```console
$ mkdir demo && cd demo
$ printf '1\n2\n3\n' > n.txt
$ shuf -n 1 n.txt; echo "exit $?"
zsh: command not found: shuf
exit 127
$ tac n.txt; echo "exit $?"
zsh: command not found: tac
exit 127
$ nproc; echo "exit $?"
zsh: command not found: nproc
exit 127
```

In bash the message is worded the other way round, same exit code:

```console
$ bash -c 'shuf; echo "exit $?"'
bash: shuf: command not found
exit 127
```

None of the three is in macOS's `/usr/bin`, and there are no `g`-prefixed
copies either unless you installed them:

```console
$ for c in shuf tac nproc gshuf gtac gnproc; do printf '%s: ' $c; command -v $c || echo none; done
shuf: none
tac: none
nproc: none
gshuf: none
gtac: none
gnproc: none
```

The replacements below use tools that ship with macOS (`sysctl` is in
`/usr/sbin`; `make`, `perl` and `python3` here come with the Command Line
Tools). I didn't
install or test GNU coreutils.

## `nproc`: the one that doesn't stop the script

`shuf` and `tac` failing is loud. `nproc` usually sits inside `$(...)`, and
there the failure turns into an empty string while the command around it
keeps going:

```console
$ echo "jobs=$(nproc)"
zsh: command not found: nproc
jobs=
```

With `make -j"$(nproc)"` that empty string leaves a bare `-j`, which make
reads as "no job limit". A Makefile with 40 targets that each wait half a
second shows it:

```console
$ mkdir mk && cd mk
$ printf 'T := $(shell jot 40)\nall: $(T)\n$(T):\n\t@perl -e "select undef, undef, undef, 0.5"\n.PHONY: all $(T)\n' > Makefile
$ /usr/bin/time -p make -j"$(nproc)"
zsh: command not found: nproc
real 0.56
user 0.07
sys 0.10
$ /usr/bin/time -p make -j"$(sysctl -n hw.logicalcpu)"
real 2.72
user 0.07
sys 0.10
$ cd ..
```

Forty half-second jobs finished in 0.56 s, so all forty ran at the same time.
With `-j8` they ran eight at a time (five rounds, 2.7 s). Here the jobs only
waited, so nothing bad happened. By the same logic, a real build with no job
limit would try to start a compiler for every ready target at once (I only
ran the sleep test). (The `real` times vary a little
from run to run; the `user`/`sys` lines too.)

### What to use instead

On this Mac (Apple M1, 4 performance + 4 efficiency cores) all of these
give 8:

```console
$ sysctl hw.ncpu hw.logicalcpu hw.physicalcpu hw.activecpu
hw.ncpu: 8
hw.logicalcpu: 8
hw.physicalcpu: 8
hw.activecpu: 8
$ getconf _NPROCESSORS_ONLN
8
$ python3 -c 'import os; print(os.cpu_count())'
8
```

I don't have a Mac where these differ, so I can't tell you which one is
"right" when they do. `sysctl -n` prints the value alone, which is what you
want inside `$(...)`:

```console
$ sysctl -n hw.logicalcpu
8
```

Apple silicon also reports the core types separately:

```console
$ sysctl hw.nperflevels hw.perflevel0.name hw.perflevel0.logicalcpu hw.perflevel1.name hw.perflevel1.logicalcpu
hw.nperflevels: 2
hw.perflevel0.name: Performance
hw.perflevel0.logicalcpu: 4
hw.perflevel1.name: Efficiency
hw.perflevel1.logicalcpu: 4
```

`hw.perflevel0` is the performance cores here. Asking for a level that
doesn't exist is an error, even with `-n`:

```console
$ sysctl -n hw.perflevel2.logicalcpu; echo "exit $?"
sysctl: unknown oid 'hw.perflevel2.logicalcpu'
exit 1
```

I only have this one Apple silicon Mac; Intel Macs weren't tested.

For a script that should also run where `nproc` exists, try it first and
fall back. The `${ncpu:?}` line stops the script if every attempt came back
empty, instead of passing a bare `-j` on:

```console
$ ncpu=$(nproc 2>/dev/null || sysctl -n hw.logicalcpu 2>/dev/null || getconf _NPROCESSORS_ONLN)
$ : "${ncpu:?could not count CPUs}"
$ echo "$ncpu"
8
```

I ran this only on this Mac, where the `nproc` branch fails and `sysctl`
answers. The Linux side wasn't tested.

## `tac`: `tail -r`

macOS `tail` has `-r` (reverse):

```console
$ tail -r n.txt
3
2
1
```

It has the same catch I hit in the
[`head -n -1` post]({% post_url 2026-10-21-macos-head-illegal-line-count-negative-drop-last-lines %}):
if the last line has no newline, `tail -r` prints it first without one, so it
runs into the next line:

```console
$ printf '1\n2\n3' > nonl.txt
$ tail -r nonl.txt | od -c
0000000    3   2  \n   1  \n
0000005
```

An awk array keeps the lines apart (and ends every line with a newline):

```console
$ awk '{a[NR]=$0} END{for(i=NR;i>0;i--) print a[i]}' nonl.txt | od -c
0000000    3  \n   2  \n   1  \n
0000006
```

The `sed -n '1!G;h;$p'` one-liner also kept them apart here, and moved the
missing newline to the end:

```console
$ sed -n '1!G;h;$p' nonl.txt | od -c
0000000    3  \n   2  \n   1
0000005
```
## `shuf`: `sort -R` is not quite a shuffle

The common answer is `sort -R`. On lines that are all different it looks
like a shuffle. With repeats, identical lines always come out next to each
other:

```console
$ printf 'a\nb\na\nc\na\nb\n' > dup.txt
$ sort -R dup.txt | tr '\n' ' '; echo
c b b a a a
$ sort -R dup.txt | tr '\n' ' '; echo
b b c a a a
```

(The order is random each time; the grouping isn't.) The man page says so:

```console
$ man sort | col -b | grep -A3 -e '-R, --random-sort'
     -R, --random-sort, --sort=random
	     Sort by a random order.  This is a random permutation of the
	     inputs except that the equal keys sort together.  It is
	     implemented by hashing the input keys and sorting the hash
```

It matters most for the `shuf -n 1` replacement. With 9 `a` lines and 1 `b`
line, `shuf -n 1` should pick `a` about 90% of the time. `sort -R | head -n 1`
picked each *distinct* line about equally in this test (two distinct
lines):

```console
$ { for i in 1 2 3 4 5 6 7 8 9; do echo a; done; echo b; } > w.txt
$ for i in $(jot 1000); do sort -R w.txt | head -n 1; done | sort | uniq -c
 501 a
 499 b
```

Fix it by making every line different first: put the line number in
front, shuffle, then cut the number off.

```console
$ for i in $(jot 1000); do awk '{print NR "\t" $0}' w.txt | sort -R | cut -f2- | head -n 1; done | sort | uniq -c
 910 a
  90 b
$ awk '{print NR "\t" $0}' dup.txt | sort -R | cut -f2- | tr '\n' ' '; echo
a b a b a c
```

Or use Perl's `List::Util` (version 1.55 with this Mac's perl 5.34.1), which
shuffles the lines themselves:

```console
$ for i in $(jot 1000); do perl -MList::Util=shuffle -e 'print +(shuffle <>)[0]' w.txt; done | sort | uniq -c
 907 a
  93 b
```

(The counts are random, so yours won't match exactly. The 50/50 vs 90/10
split is the point.)

The short `perl -e 'print shuffle <>'` has the same missing-newline problem
as `tail -r`, whenever the unterminated line doesn't land last. Strip the
newlines and add them back (both orders below are random):

```console
$ perl -MList::Util=shuffle -e 'print shuffle <>' nonl.txt | od -c
0000000    1  \n   3   2  \n
0000005
$ perl -MList::Util=shuffle -e 'chomp(my @l = <>); print map { "$_\n" } shuffle @l' nonl.txt | od -c
0000000    1  \n   3  \n   2  \n
0000006
```

### Random numbers: `jot`

For `shuf -i 1-10 -n 3` (three different numbers from 1 to 10), shuffle the
range with `jot`. `jot -r` draws each number on its own, so it can repeat:

```console
$ jot 10 | sort -R | head -n 3 | tr '\n' ' '; echo
4 5 3
$ jot -r 5 1 3 | tr '\n' ' '; echo
1 1 3 2 3
```

### Don't seed awk with `srand()` in a loop

An awk shuffle using `srand()` with no argument: three runs in the
same second gave the same "random" number:

```console
$ for i in 1 2 3; do awk 'BEGIN{srand(); print rand()}'; done
0.0896271
0.0896271
0.0896271
```

The number itself will be different when you run it; the repeat is the point.
Use one of the shuffles above instead.

### The same shuffle twice

If you want a repeatable order (tests, a fixed sample), `sort -R` takes a
seed file:

```console
$ printf 'seed-1\n' > seed
$ jot 6 | sort -R --random-source=seed | tr '\n' ' '; echo
3 4 5 1 2 6
$ jot 6 | sort -R --random-source=seed | tr '\n' ' '; echo
3 4 5 1 2 6
```

The equal-lines-together rule still applies.

## Checklist

- `zsh: command not found: shuf` / `tac` / `nproc`, exit 127: none of them
  ships with macOS.
- `$(nproc)` fails quietly: `make -j"$(nproc)"` becomes `make -j`, no job
  limit. Use `sysctl -n hw.logicalcpu` or `getconf _NPROCESSORS_ONLN`, and
  guard with `: "${ncpu:?}"`.
- Performance cores only: `sysctl -n hw.perflevel0.logicalcpu` (Apple
  silicon).
- `tac` → `tail -r`. Without a final newline it glues the last two lines;
  the awk array version doesn't.
- `shuf` → `sort -R`, but identical lines stay together, and
  `sort -R | head -n 1` picked a and b about equally (9:1 input). Prefix line
  numbers (`awk '{print NR "\t" $0}' | sort -R | cut -f2-`) or use Perl's
  `List::Util` shuffle with `chomp`.
- `shuf -i` → `jot N | sort -R | head`; `jot -r` repeats numbers.
- awk `srand()` gave the same number in three runs within one second.
- Repeatable order: `sort -R --random-source=FILE`.

*Tested on macOS 26.5.2 (Apple M1) in zsh 5.9 (arm64-apple-darwin25.0) with
`/usr/bin/tail` and `/usr/bin/sort` (`text_cmds-199`, sort "2.3-Apple
(199)"), `/usr/bin/jot` (`shell_cmds-329`), `/usr/sbin/sysctl` and
`/usr/bin/getconf` (`system_cmds-1042.120.1`), `/usr/bin/awk` (awk version
20200816), GNU Make 3.81, perl 5.34.1 (List::Util 1.55), Python 3.8.2; the
bash line with bash 3.2.57. GNU coreutils, Linux and Intel Macs were not
tested.*
