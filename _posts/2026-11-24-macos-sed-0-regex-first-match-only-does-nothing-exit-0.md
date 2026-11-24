---
title: "macOS sed `0,/regex/` does nothing (exit 0) — replace only the first match, and `0,/^$/!d` empties the file"
date: 2026-11-24
description: >-
  On macOS, /usr/bin/sed accepts the GNU idiom `sed '0,/foo/s//bar/'`
  (replace only the first match) without an error, changes nothing, and
  exits 0. Line 0 never exists, so the range never starts. With `sed -i ''`
  the file is still rewritten (new inode, new mtime), so a version-bump
  script prints "bumped" while the old version stays. `sed -n '0,/^$/p'`
  prints nothing and `sed -i '' '0,/^$/!d'` leaves a 0-byte file. `1,/re/`
  is wrong when line 1 matches, and the one-line loop `{s//bar/;:a;n;ba;}`
  fails with "unexpected EOF (pending }'s)". Fixes: split the loop with -e,
  awk, `/re/q`, and check the result.
tags: [macos, sed, shell, awk, bsd]
---

"Replace only the first match" with sed, on a Mac:

```console
$ printf 'a\nfoo\nb\nfoo\n' | sed '0,/foo/s//bar/'; echo "exit $?"
a
foo
b
foo
exit 0
```

Nothing was replaced, and there is no error. `0,/regex/` is a GNU sed
address: the GNU manual describes it as an extension that lets the regex
match on the very first line, so the range ends at the first match. I did
not run GNU sed for this post; the expected result there is `bar` on line 2
only. On macOS, `/usr/bin/sed` (text_cmds-199) does not reject it.

## Line 0 never matches, and sed does not say so

macOS sed counts lines from 1. An address of 0 is accepted, matches no line,
and every command behind it is skipped:

```console
$ printf 'one\ntwo\n' | sed '0d'; echo "exit $?"
one
two
exit 0
$ printf 'one\ntwo\n' | sed -n '0p'; echo "exit $?"
exit 0
$ printf 'one\ntwo\n' | sed '0,1d'; echo "exit $?"
one
two
exit 0
```

A range starts "with the first pattern space that matches the first
address" (`man sed`, "Sed Addresses"). The first address is line 0, so
`0,/foo/` never starts, and the `s` never runs. Exit status 0 only means
sed read its input and wrote its output.

## A version bump that prints "bumped" and changes nothing

A release step sets the `[project]` version in `pyproject.toml`. There is a
second `version =` line further down that must stay as it is, so the
script changes only the first one. It came from Linux; the `''` after `-i`
was added for macOS (see
[macOS `sed -i`](/2026/10/macos-sed-i-command-c-expects-invalid-command-code/)),
the `0,` address was left alone.

Everything below runs in a scratch folder:

```console
$ T=$(mktemp -d)
$ cd "$T"
$ cat > pyproject.toml <<'EOF'
> [project]
> name = "notes-sync"
> version = "1.2.3"
> requires-python = ">=3.9"
> 
> [tool.release]
> version = "1.0"
> EOF
$ cat > bump.sh <<'EOF'
> #!/bin/sh
> # set the [project] version: the first "version = " line only
> new=${1:?usage: bump.sh NEW_VERSION}
> sed -i '' "0,/^version = /s/^version = .*/version = \"$new\"/" pyproject.toml
> echo "bumped to $new"
> EOF
$ stat -f '%i %Sm' -t %T pyproject.toml
20891089 10:02:16
$ sleep 1
$ sh bump.sh 1.3.0; echo "exit $?"
bumped to 1.3.0
exit 0
$ grep -n '^version' pyproject.toml
3:version = "1.2.3"
7:version = "1.0"
$ stat -f '%i %Sm' -t %T pyproject.toml
20891093 10:02:18
```

(The inode numbers and times are from my run and will differ on yours.)

The script says "bumped to 1.3.0" and exits 0; the file still says 1.2.3.
`sed -i` wrote a new file anyway: the inode number and the modification
time both changed. So a check like "was pyproject.toml modified?" or a
newer mtime does not catch this either; only the content tells.

## The other direction: `0,/^$/!d` deletes everything

The same address is used to keep "everything up to the first blank line",
for example the header block of a mail or a front-matter file. With `!`,
the command runs on every line *outside* the range, and here every line is
outside it:

```console
$ printf 'Subject: hi\nFrom: me\n\nbody\nmore\n' > mail.txt
$ sed -n '0,/^$/p' mail.txt; echo "exit $?"
exit 0
$ cp mail.txt header.txt
$ sed -i '' '0,/^$/!d' header.txt; echo "exit $?"
exit 0
$ wc -c mail.txt header.txt
      32 mail.txt
       0 header.txt
      32 total
```

`-n ... p` prints nothing; `!d` with `-i` leaves a 0-byte file, exit 0.

## `1,/re/` works until line 1 matches

The usual first try is to change `0` to `1`. That gives the same result
when the first match is on line 2 or later:

```console
$ printf 'a\nfoo\nb\nfoo\n' | sed '1,/foo/s/foo/bar/'
a
bar
b
foo
$ printf 'foo\na\nfoo\nb\n' | sed '1,/foo/s/foo/bar/'
bar
a
bar
b
```

When line 1 itself matches, both matches are replaced. The man page says
why: "In the case when the second address is a context address, sed does
not re-match the second address against the pattern space that matched the
first address." Line 1 starts the range, and the end is looked for only
from line 2, so the range runs to the *second* `foo`. Avoiding that case is
exactly what GNU's `0,` is for.

## The one-line loop fails on macOS sed

The other well-known answer is a loop: on the first match, substitute,
then print and read every remaining line without testing it. Written on one
line, macOS sed rejects it:

```console
$ printf 'a\nfoo\nb\nfoo\n' | sed '/foo/{s//bar/;:a;n;ba;}'; echo "exit $?"
sed: 2: "/foo/{s//bar/;:a;n;ba;}
": unexpected EOF (pending }'s)
exit 1
```

The error shows that this sed reads the label name to the end of the line, so
`:a;n;ba;}` behaves as one label with a long name, and the `}` is never seen. With `-i` this fails
before writing, so at least it is loud (exit 1).

## What works

Split the loop with `-e`, so the label and the `b` each end a line:

```console
$ printf 'a\nfoo\nb\nfoo\n' | sed -e '/foo/{s//bar/;:a' -e 'n;ba' -e '}'; echo "exit $?"
a
bar
b
foo
exit 0
$ printf 'foo\na\nfoo\nb\n' | sed -e '/foo/{s//bar/;:a' -e 'n;ba' -e '}'
bar
a
foo
b
$ printf 'a\nb\n' | sed -e '/foo/{s//bar/;:a' -e 'n;ba' -e '}'
a
b
```

Only the first `foo` changes, also when it is on line 1. With no match the
input comes out unchanged (and exit 0, so you still have to check).

awk says the same thing more readably:

```console
$ printf 'foo\na\nfoo\nb\n' | awk '!done && sub(/foo/, "bar") { done = 1 } 1'
bar
a
foo
b
```

awk has no in-place option on macOS; write to a temporary file and move it
back, and check the moved file's permissions (a `mktemp` file is `-rw-------`).

For "everything up to and including the first match", `q` is enough:

```console
$ sed '/^$/q' mail.txt
Subject: hi
From: me

```

And for "delete through the first match" (GNU `0,/re/d`), `1,/re/d` has the
line-1 problem again; awk does not:

```console
$ printf 'foo\na\nfoo\nb\n' | sed '1,/foo/d'
b
$ printf 'foo\na\nfoo\nb\n' | awk 'done; /foo/ { done = 1 }'
a
foo
b
```

## The fixed bump script

Use the split loop, reject versions that would break the `s` command, and
read the file back before saying "bumped":

```console
$ cat > bump.sh <<'EOF'
> #!/bin/sh
> # set the [project] version: the first "version = " line only
> new=${1:?usage: bump.sh NEW_VERSION}
> case $new in
>   ''|*[!0-9.]*) echo "bad version: $new" >&2; exit 2 ;;
> esac
> sed -i '' -e "/^version = /{s/^version = .*/version = \"$new\"/;:a" -e 'n;ba' -e '}' pyproject.toml || exit 1
> got=$(grep -m1 '^version = ' pyproject.toml)
> if [ "$got" != "version = \"$new\"" ]; then
>   echo "pyproject.toml not changed: $got" >&2
>   exit 1
> fi
> echo "bumped to $new"
> EOF
$ sh bump.sh 1.3.0; echo "exit $?"
bumped to 1.3.0
exit 0
$ grep -n '^version' pyproject.toml
3:version = "1.3.0"
7:version = "1.0"
$ sh bump.sh 1.3.x; echo "exit $?"
bad version: 1.3.x
exit 2
$ printf '[project]\nname = "x"\n' > pyproject.toml
$ sh bump.sh 1.4.0; echo "exit $?"
pyproject.toml not changed: 
exit 1
```

The last run is a file with no `version =` line: sed changes nothing and
exits 0, and the read-back check is what stops the script. The same check
would have caught the original `0,` version.

Clean up the scratch folder:

```console
$ cd /
$ rm -r "${T:?}"
```

## Checklist

- macOS `/usr/bin/sed` accepts line address `0` (`0d`, `0p`, `0,/re/`),
  matches nothing and exits 0. GNU's "first match only" `0,/re/` does
  nothing here.
- With `-i ''` the file is rewritten anyway (new inode, new mtime), so an
  mtime check does not show that nothing changed.
- `0,/re/!d` and `-n '0,/re/p'` turn into "delete everything" / "print
  nothing"; with `-i` that is a 0-byte file.
- `1,/re/` differs from `0,/re/` when line 1 matches: the range then runs
  to the second match.
- macOS sed reads a label to the end of the line: `:a;n;ba;}` on one line is
  "unexpected EOF (pending }'s)", exit 1. Split it:
  `sed -e '/re/{s//new/;:a' -e 'n;ba' -e '}'`.
- Alternatives: `awk '!done && sub(/re/, "new") { done = 1 } 1'`,
  `sed '/re/q'` (up to the first match), `awk 'done; /re/ { done = 1 }'`
  (after the first match).
- macOS `/usr/bin/sed` exits 0 whether or not anything matched. Read the result back before
  reporting success.

Related: [macOS `sed -i`](/2026/10/macos-sed-i-command-c-expects-invalid-command-code/)
and [macOS sed: `\s`, `\d`, `\+` not working](/2026/11/macos-sed-s-d-plus-not-working-trailing-whitespace-eats-letters/).

*Tested on 2026-11-24 on macOS 26.5.2 (Apple silicon, APFS) with
`/usr/bin/sed` (text_cmds-199), `/usr/bin/awk` (version 20200816),
`/usr/bin/grep` (2.6.0-FreeBSD) and `/usr/bin/stat`, zsh 5.9 and `/bin/sh` (bash 3.2.57),
in `env -i` with `PATH=/usr/bin:/bin:/usr/sbin:/sbin`.
Not tested: GNU sed (not installed here; its `0,/re/` behaviour is from
the GNU manual), Linux, older macOS versions.*
