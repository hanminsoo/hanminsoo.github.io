---
title: "macOS `numfmt: command not found` (exit 127) — and the size limit that uploads every file"
description: >-
  macOS has no `numfmt`: `numfmt --to=iec` and `numfmt --from=iec 2M` fail
  with "command not found" and exit 127. In a script, `max=$(numfmt
  --from=iec 2M)` is empty, `[ "$bytes" -gt "$max" ]` prints "integer
  expression expected" and every file passes the size limit, exit 0.
  `set -e` stops the script at that assignment but not at `echo "$(numfmt
  ...)"`, which prints a blank size. A naive awk replacement prints
  "1024.0K" for 1048575 bytes. Fix: two small awk functions, human and
  to_bytes, that check their input and fail loudly.
tags: [macos, numfmt, shell, awk, bsd]
---

Turning bytes into "2.9M" (or "2M" into bytes) with `numfmt`, on a Mac:

```console
$ numfmt --to=iec 123456789; echo "exit $?"
zsh: command not found: numfmt
exit 127
$ bash -c 'numfmt --from=iec 2M'; echo "exit $?"
bash: numfmt: command not found
exit 127
```

`numfmt` is part of GNU coreutils. On this Mac (macOS 26.5.2, Homebrew's
coreutils not installed) it is not in any of the system bin folders and has
no man page:

```console
$ command -v numfmt; echo "exit $?"
exit 1
$ ls /bin/numfmt /sbin/numfmt /usr/bin/numfmt /usr/sbin/numfmt
ls: /bin/numfmt: No such file or directory
ls: /sbin/numfmt: No such file or directory
ls: /usr/bin/numfmt: No such file or directory
ls: /usr/sbin/numfmt: No such file or directory
$ man numfmt; echo "exit $?"
No manual entry for numfmt
exit 1
```

Exit 127 means "command not found". On a terminal that is obvious; in a
script that was written on Linux, it turns into wrong numbers.

## The size limit that lets every file through

A small upload step: skip any backup larger than `MAX` (default 2M).

Everything below runs in a scratch folder:

```console
$ T=$(mktemp -d)
$ cd "$T"
$ mkdir backups
$ head -c 3000000 /dev/zero > backups/db.dump
$ head -c 1500000 /dev/zero > backups/site.tar
$ cat limit.sh
#!/bin/sh
# refuse to upload a backup over MAX (default 2M)
max=$(numfmt --from=iec "${MAX:-2M}")
for f in backups/*; do
  bytes=$(stat -f %z "$f")
  if [ "$bytes" -gt "$max" ]; then
    echo "skip $f: $bytes > $max" >&2
    continue
  fi
  echo "upload $f"
done
$ sh limit.sh; echo "exit $?"
limit.sh: line 3: numfmt: command not found
limit.sh: line 6: [: : integer expression expected
upload backups/db.dump
limit.sh: line 6: [: : integer expression expected
upload backups/site.tar
exit 0
```

`db.dump` is 3,000,000 bytes, over the intended 2 MiB (2,097,152-byte) limit, and it is
"uploaded" anyway; the script exits 0.

- `numfmt` is not found, so `$(...)` captures nothing and `max` is empty.
  The error goes to stderr; the assignment's status is 127, but nothing
  checks it.
- `[ 3000000 -gt "" ]` is an error ("integer expression expected"). An error
  inside `if` counts as false, so the script goes on to the upload line.

## `set -e` catches the assignment, not the `echo`

`set -e` does stop at a plain assignment whose command substitution fails:

```console
$ sh -ec 'max=$(numfmt --from=iec 2M); echo "max=$max"'; echo "exit $?"
sh: numfmt: command not found
exit 127
$ sh -ec 'echo "max=$(numfmt --from=iec 2M)"'; echo "exit $?"
sh: numfmt: command not found
max=
exit 0
```

The second line is the same failure, but the substitution is an argument of
`echo`. Its status is lost; `echo` succeeds, so `set -e` sees 0. A report
script written that way prints blank sizes and finishes normally:

```console
$ cat report.sh
#!/bin/sh
# print each backup's size in a human-readable form
set -e
for f in backups/*; do
  bytes=$(stat -f %z "$f")
  echo "$f: $(numfmt --to=iec "$bytes")"
done
echo "report done"
$ sh report.sh; echo "exit $?"
report.sh: line 6: numfmt: command not found
backups/db.dump: 
report.sh: line 6: numfmt: command not found
backups/site.tar: 
report done
exit 0
```

## What macOS has instead

For files, `du -Ah` prints a human-readable apparent size (powers of 1024,
see [macOS `du -b`](/2026/11/macos-du-b-invalid-option-size-in-bytes-apparent-size/)):

```console
$ du -Ah backups/*
2.9M	backups/db.dump
1.4M	backups/site.tar
```

That only works on files, not on a number you already have, and I found
nothing in the base system that goes the other way ("2M" to bytes). For that you need your own
code. `awk` is in the base system, but the first version that comes to mind
has an edge:

```console
$ printf '%s\n' 1023 1536 1048575 3000000 | awk '{ n = $1; u = "BKMGTPE"; i = 1
>   while (n >= 1024 && i < 7) { n /= 1024; i++ }
>   printf "%s -> %.1f%s\n", $1, n, substr(u, i, 1) }'
1023 -> 1023.0B
1536 -> 1.5K
1048575 -> 1024.0K
3000000 -> 2.9M
```

1,048,575 bytes is 1023.999 KiB. It is below 1024, so the loop stops at K,
and then `%.1f` rounds it up to "1024.0K". The unit has to be chosen after
rounding, not before.

## Two awk functions: `human` and `to_bytes`

```console
$ cat size.sh
# size.sh: numfmt --to=iec / --from=iec stand-ins for macOS (source it)

# human 3000000 -> 2.9M   (powers of 1024, one decimal, bytes as 123B)
human() {
  awk -v n="$1" 'BEGIN {
    if (n !~ /^[0-9]+$/) { print "human: not a byte count: " n > "/dev/stderr"; exit 1 }
    u = "BKMGTPE"; i = 1
    if (n < 1024) { printf "%dB\n", n; exit }
    while (i < 7 && sprintf("%.1f", n) + 0 >= 1024) { n /= 1024; i++ }
    printf "%.1f%s\n", n, substr(u, i, 1)
  }'
}

# to_bytes 2M -> 2097152   (K M G T = powers of 1024; optional "i" or "B")
to_bytes() {
  awk -v s="$1" 'BEGIN {
    t = toupper(s); sub(/I?B$/, "", t)
    if (t !~ /^[0-9]+(\.[0-9]+)?[KMGT]?$/) { print "to_bytes: bad size: " s > "/dev/stderr"; exit 1 }
    m = 1; c = substr(t, length(t), 1)
    if (c ~ /[KMGT]/) { m = 1024 ^ index("KMGT", c); t = substr(t, 1, length(t) - 1) }
    printf "%.0f\n", t * m
  }'
}
$ . ./size.sh
$ for n in 0 1023 1024 1536 1048575 3000000 123456789; do echo "$n -> $(human $n)"; done
0 -> 0B
1023 -> 1023B
1024 -> 1.0K
1536 -> 1.5K
1048575 -> 1.0M
3000000 -> 2.9M
123456789 -> 117.7M
$ human 12abc; echo "exit $?"
human: not a byte count: 12abc
exit 1
$ human ""; echo "exit $?"
human: not a byte count: 
exit 1
$ for s in 2M 2m 2MiB 2MB 1.5K 512 1G; do echo "$s -> $(to_bytes $s)"; done
2M -> 2097152
2m -> 2097152
2MiB -> 2097152
2MB -> 2097152
1.5K -> 1536
512 -> 512
1G -> 1073741824
$ to_bytes 2X; echo "exit $?"
to_bytes: bad size: 2X
exit 1
$ to_bytes ""; echo "exit $?"
to_bytes: bad size: 
exit 1
```

Notes on these two:

- They are not a copy of GNU `numfmt`. I did not run GNU `numfmt` for this
  post, so I have not compared its rounding or its suffix rules with these.
  Here, "2M", "2MiB" and "2MB" all mean 2 × 1024 × 1024; if your sizes are
  powers of 1000, change the 1024s.
- The point is the `exit 1` on bad input, so the caller can stop: an empty
  `MAX` or a typo like `2X` must not turn into "no limit".
- awk numbers are doubles, so very large counts lose precision. With this
  Mac's `/usr/bin/awk`, 2^53 (8192T) still comes out exact, one byte more
  does not:

```console
$ for s in 8192T 9007199254740992 9007199254740993 9007199254740995; do echo "$s -> $(to_bytes $s)"; done
8192T -> 9007199254740992
9007199254740992 -> 9007199254740992
9007199254740993 -> 9007199254740992
9007199254740995 -> 9007199254740996
```

That is far beyond any file size here, so I did not add a check for it.

## The fixed scripts

Check every substitution's status where it is assigned, and do not hide a
lookup inside `echo` arguments unless its input is already known to be good:

```console
$ cat limit.sh
#!/bin/sh
# refuse to upload a backup over MAX (default 2M)
. "$(dirname "$0")/size.sh"
max=$(to_bytes "${MAX:-2M}") || exit 2
for f in backups/*; do
  bytes=$(stat -f %z "$f") || exit 2
  if [ "$bytes" -gt "$max" ]; then
    echo "skip $f: $(human "$bytes") ($bytes bytes), limit ${MAX:-2M} ($max bytes)" >&2
    continue
  fi
  echo "upload $f ($(human "$bytes"))"
done
$ sh limit.sh; echo "exit $?"
skip backups/db.dump: 2.9M (3000000 bytes), limit 2M (2097152 bytes)
upload backups/site.tar (1.4M)
exit 0
$ MAX=1G sh limit.sh; echo "exit $?"
upload backups/db.dump (2.9M)
upload backups/site.tar (1.4M)
exit 0
$ MAX=2X sh limit.sh; echo "exit $?"
to_bytes: bad size: 2X
exit 2
$ MAX=1.4M sh limit.sh; echo "exit $?"
skip backups/db.dump: 2.9M (3000000 bytes), limit 1.4M (1468006 bytes)
skip backups/site.tar: 1.4M (1500000 bytes), limit 1.4M (1468006 bytes)
exit 0
$ cat report.sh
#!/bin/sh
# print each backup's size in a human-readable form
set -e
. "$(dirname "$0")/size.sh"
for f in backups/*; do
  bytes=$(stat -f %z "$f")
  h=$(human "$bytes")
  echo "$f: $h"
done
echo "report done"
$ sh report.sh; echo "exit $?"
backups/db.dump: 2.9M
backups/site.tar: 1.4M
report done
exit 0
```

The `MAX=1.4M` run shows why the skip message prints bytes too: 1,500,000
bytes and the 1,468,006-byte limit both round to "1.4M".

In `limit.sh` the two `$(human ...)` calls inside `echo` are the pattern the
section above warns about; they are left that way because `bytes` and `max`
have already been checked (a failed `stat` exits, a bad `MAX` exits 2).

Clean up the scratch folder:

```console
$ cd /
$ rm -r "${T:?}"
```

## Checklist

- macOS 26.5.2 here has no `numfmt` (not in `/bin`, `/sbin`, `/usr/bin`,
  `/usr/sbin`; no man page). Calling it is "command not
  found", exit 127.
- `x=$(numfmt ...)` leaves `x` empty; `[ "$n" -gt "$x" ]` then errors, the
  `if` skips its `then` block and the script goes on to the upload line, so a
  size limit lets everything through.
- `set -e` stops at `x=$(numfmt ...)` (exit 127) but not at
  `echo "$(numfmt ...)"`, which prints a blank value and exits 0.
- For files, `du -Ah` gives human-readable sizes. For a number, or for "2M"
  to bytes, use your own function.
- In an awk formatter, pick the unit after rounding, or 1,048,575 bytes
  becomes "1024.0K".
- Make the replacement fail (non-zero exit) on bad input, and check it:
  `max=$(to_bytes "$MAX") || exit 2`.

Related: [macOS `du -b`](/2026/11/macos-du-b-invalid-option-size-in-bytes-apparent-size/)
and [macOS `stat: illegal option -- c`](/2026/10/macos-stat-illegal-option-c-file-size-mtime-wc-padding/).

*Tested on 2026-11-23 on macOS 26.5.2 (Apple silicon, APFS) with `/usr/bin/awk`
(version 20200816, awk-40), `/usr/bin/stat` and `/usr/bin/du` (file_cmds-479),
zsh 5.9 and `/bin/sh` (bash 3.2.57), in `env -i` with
`PATH=/usr/bin:/bin:/usr/sbin:/sbin`.
Not tested: GNU coreutils `numfmt` (Homebrew's coreutils is not installed
here), Linux, older macOS versions, other `sh` implementations.*
