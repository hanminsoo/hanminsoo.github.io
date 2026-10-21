---
title: "macOS `awk: calling undefined function gensub`: replace it with match/substr, `sed -E`, or `perl -pe` (and why it can pass silently)"
description: >-
  On macOS, `gensub()` fails with "awk: calling undefined function gensub"
  (exit 2) because /usr/bin/awk is the BSD "one true awk" (version 20200816),
  not gawk. In my tests the error only appeared when a line actually reached
  the gensub call: earlier lines were already printed, input without a match
  passed with exit 0, and a pipe hid the exit 2. Use gsub when you don't need
  backreferences, a match/substr loop when you do, or `sed -E` / `perl -pe`.
  Reproduced on macOS 26.
tags: [macos, shell, awk, bsd]
---

An awk one-liner that swaps date parts stops on a Mac:

```console
$ printf 'a\nb\nc 2026-10-21\nd\n' > log.txt
$ awk '/-/ { $0 = gensub(/([0-9]+)-([0-9]+)-([0-9]+)/, "\\3/\\2/\\1", "g") } { print }' log.txt; echo "exit $?"
a
b
awk: calling undefined function gensub
 input record number 3, file log.txt
 source line number 1
exit 2
```

`gensub` is a gawk (GNU awk) function, according to gawk's manual; I didn't
run gawk here. macOS ships a different awk:

```console
$ awk --version
awk version 20200816
$ what /usr/bin/awk
/usr/bin/awk:
	PROGRAM:awk  PROJECT:awk-40
	PROGRAM:awk  PROJECT:awk-40
```

## The error comes late, or not at all

Look at the output above again: `a` and `b` were printed before the error.
This awk did not reject the script when it read it. It failed when line 3
was the first line to reach `gensub`.

So the same script passes when no line reaches the call:

```console
$ printf 'a\nb\n' | awk '/-/ { $0 = gensub(/x/, "y", "g") } { print }'; echo "exit $?"
a
b
exit 0
```

A script tested on input without dates would look fine on this Mac. And in
a pipe, the exit status you see is the last command's:

```console
$ awk '/-/ { $0 = gensub(/([0-9]+)-([0-9]+)-([0-9]+)/, "\\3/\\2/\\1", "g") } { print }' log.txt | cat; echo "exit $?"
awk: calling undefined function gensub
 input record number 3, file log.txt
 source line number 1
a
b
exit 0
```

Output stopped at line 2, and the pipeline still says 0. (`set -o pipefail`
in bash or zsh makes a failing awk fail the pipeline.)

Two other gawk-isms failed the same way here. `strftime` gives the same
"calling undefined function" at run time; the three-argument `match(s, re, arr)`
is a syntax error before any input is read:

```console
$ echo 'c 2026-10-21' | awk '{ print strftime("%Y") }'; echo "exit $?"
awk: calling undefined function strftime
 input record number 1, file 
 source line number 1
exit 2
$ echo 'c 2026-10-21' | awk '{ if (match($0, /([0-9]+)-([0-9]+)/, m)) print m[1] }'; echo "exit $?"
awk: syntax error at source line 1
 context is
	{ if (match($0, >>>  /([0-9]+)-([0-9]+)/, <<< 
awk: illegal statement at source line 1
awk: illegal statement at source line 1
exit 2
```

## No backreferences needed: `gsub`

If the replacement doesn't use `\\1`, `\\2`…, `gsub` does the job. It
changes its third argument in place (or `$0`) and returns the number of
replacements, so copy first if you need the original:

```console
$ echo 'a.b.c' | awk '{ s = $0; n = gsub(/\./, "_", s); print n, s, $0 }'
2 a_b_c a.b.c
```

`&` in the replacement is the matched text, and `\\&` is a literal `&`:

```console
$ echo 'cat bat' | awk '{ gsub(/[cb]at/, "<&>"); print }'
<cat> <bat>
$ echo 'cat bat' | awk '{ gsub(/[cb]at/, "\\&"); print }'
& &
```

## Backreferences in awk: a `match` / `substr` loop

This awk has no capture groups. Match the whole piece, cut it out with
`substr`, and take it apart yourself (here with `split`). Save as `swap.awk`:

```awk
{
  out = ""
  s = $0
  while (match(s, /[0-9]+-[0-9]+-[0-9]+/)) {
    split(substr(s, RSTART, RLENGTH), p, "-")
    out = out substr(s, 1, RSTART - 1) p[3] "/" p[2] "/" p[1]
    s = substr(s, RSTART + RLENGTH)
  }
  print out s
}
```

```console
$ printf 'start 2026-10-21 end 2025-01-02\nno date here\n' > dates.txt
$ awk -f swap.awk dates.txt; echo "exit $?"
start 21/10/2026 end 02/01/2025
no date here
exit 0
```

`match` sets `RSTART` (where the match starts) and `RLENGTH` (its length).
The loop keeps the text before each match, adds the rewritten date, and
continues on the rest of the line.

**The regex must not match an empty string.** If it can, `match` succeeds
with `RLENGTH` 0, `s` never gets shorter, and the loop would never end. With a
counter capping it at 5 rounds:

```console
$ echo 'ab12' | awk '{ s = $0; n = 0; while (match(s, /[0-9]*/) && n < 5) { n++; print n, RSTART, RLENGTH; s = substr(s, RSTART + RLENGTH) } }'
1 1 0
2 1 0
3 1 0
4 1 0
5 1 0
```

Use `+` rather than `*` for the part that must be there.

## Or leave awk: `sed -E` and `perl -pe`

If the substitution is the whole job, macOS's BSD sed does it with `-E`
(unescaped groups and `\1` backreferences):

```console
$ sed -E 's|([0-9]+)-([0-9]+)-([0-9]+)|\3/\2/\1|g' dates.txt; echo "exit $?"
start 21/10/2026 end 02/01/2025
no date here
exit 0
```

and so does perl (`$1`, `$2`…):

```console
$ perl -pe 's|(\d+)-(\d+)-(\d+)|$3/$2/$1|g' dates.txt; echo "exit $?"
start 21/10/2026 end 02/01/2025
no date here
exit 0
```

`|` as the delimiter avoids escaping the `/` in the replacement.

gensub's "replace only the Nth match" form (a number instead of `"g"`) has
a sed equivalent, a number after the last delimiter:

```console
$ echo 'x x x' | sed 's/x/y/2'
x y x
```

## Checklist

- `awk: calling undefined function gensub` (exit 2): `/usr/bin/awk` on macOS
  is awk version 20200816 (`awk-40`), not gawk.
- In my tests the error came only when a line reached the call. Lines before
  it were already printed; input that never reached it passed with exit 0.
  Test with input that hits every branch.
- In a pipe the error is hidden behind the last command's exit 0 — use
  `set -o pipefail`, or check the awk step alone.
- No backreferences: `gsub(re, repl[, target])`; `&` is the match, `\\&` a
  literal `&`. It edits in place and returns a count.
- Backreferences: a `while (match(...))` loop with `substr`, `RSTART`,
  `RLENGTH`. The regex must not be able to match an empty string.
- Just substituting: `sed -E 's|(..)|\1|g'` or `perl -pe 's|(..)|$1|g'`;
  Nth match: `sed 's/x/y/2'`.
- Same family of failures here: `strftime` (undefined function), and
  `match(s, re, arr)` (syntax error).

*Tested on macOS 26.5.2 (Apple silicon) with `/usr/bin/awk`
(awk version 20200816, `PROGRAM:awk  PROJECT:awk-40`), `/usr/bin/sed`
(`text_cmds-199`) and perl 5.34.1, in zsh 5.9 (arm64-apple-darwin25.0).
The `pipefail` line was checked in zsh 5.9 and bash 3.2.57 (pipeline exit
2 in both); everything else ran in zsh only. gawk and Linux were not tested.*
