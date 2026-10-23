---
title: "macOS `sha256sum -c` prints `usage: sha256sum [-bctwz] [files ...]`: pass `-` for stdin, and empty input passes with exit 0"
description: >-
  On macOS 26, `echo "$hash  file" | sha256sum -c` prints
  "usage: sha256sum [-bctwz] [files ...]" and exits 1 — the built-in
  sha256sum (/sbin/sha256sum, the md5 program in GNU mode) needs `-` to read
  the checksum from stdin. Other traps: empty input and `--ignore-missing`
  with no matching file both exit 0 silently, a malformed line is skipped
  with exit 0 unless you add `--strict`, CRLF sums files fail with
  "No such file or directory", and without /sbin on PATH it's
  "command not found". The `shasum -a 256` fallback and how it differs.
tags: [macos, shell, sha256sum, shasum, checksum, bsd]
---

The usual one-liner for checking a download against a published hash, on a
Mac:

```console
$ mkdir demo && cd demo
$ printf 'hello\n' > a.txt
$ printf 'world\n' > b.txt
$ h=5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03
$ echo "$h  a.txt" | sha256sum -c; echo "exit $?"
usage: sha256sum [-bctwz] [files ...]
exit 1
```

Give it `-` as the file to check, and it reads the line from stdin:

```console
$ echo "$h  a.txt" | sha256sum -c -; echo "exit $?"
a.txt: OK
exit 0
```

`/dev/stdin` works too. Its man page says "If no files are listed on the
command line, or a file name is given as -, input is taken from stdin" — for
hashing that's true (`sha256sum < a.txt` works), but with `-c` and no file
it printed the usage line here.

## Which `sha256sum` this is

It isn't GNU coreutils. On this macOS 26 it's in `/sbin`, a hard link to
`md5` (same inode), which switches to GNU-style options and output when its
name ends in `sum`:

```console
$ command -v sha256sum
/sbin/sha256sum
$ ls -i /sbin/md5 /sbin/sha256sum | awk '{print $1}' | uniq | wc -l
       1
$ sha256sum --version
sha256sum (Darwin) 1.0
$ sha256sum a.txt b.txt
5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03  a.txt
e258d248fda94c63753607f7c4494ee0fcbe92f1a76bfdac795c9d84101eb317  b.txt
```

`md5sum`, `sha1sum`, `sha224sum`, `sha384sum` and `sha512sum` sit next to
it. I only have this macOS 26 machine, so I can't say which macOS version
added them.

### `command not found: sha256sum`

Because it's in `/sbin`, anything that runs with a shorter `PATH` doesn't
see it. Terminal gets `/sbin` from `/etc/paths`; a job started with
`PATH=/usr/bin:/bin` doesn't:

```console
$ env PATH=/usr/bin:/bin sha256sum a.txt; echo "exit $?"
env: sha256sum: No such file or directory
exit 127
$ env PATH=/usr/bin:/bin zsh -fc 'sha256sum a.txt'; echo "exit $?"
zsh:1: command not found: sha256sum
exit 127
```

Call it as `/sbin/sha256sum`, add `/sbin` to that job's `PATH`, or use
`shasum -a 256` from `/usr/bin` (below).

## Empty input: no output, exit 0

A common way to check just the file you downloaded is to pull its line out
of the full `SHA256SUMS`. If the name doesn't match — a typo, a different
version — `grep` finds nothing, `sha256sum -c -` gets empty input, prints
nothing and exits 0. `--strict` doesn't change that:

```console
$ sha256sum a.txt b.txt > SHA256SUMS
$ grep ' a-1.2.txt$' SHA256SUMS | sha256sum -c -; echo "exit $?"
exit 0
$ : | sha256sum -c --strict -; echo "exit $?"
exit 0
```

`--ignore-missing` has the same shape: if none of the listed files are
there, nothing is verified and it still exits 0:

```console
$ echo "$h  other.iso" | sha256sum -c -; echo "exit $?"
sha256sum:  other.iso: No such file or directory
exit 1
$ echo "$h  other.iso" | sha256sum -c --ignore-missing -; echo "exit $?"
exit 0
```

So "exit 0" alone doesn't prove anything was checked. Either look for the
`name: OK` line:

```console
$ grep ' a.txt$' SHA256SUMS | sha256sum -c - | grep -qx 'a.txt: OK' && echo verified
verified
$ grep ' a-1.2.txt$' SHA256SUMS | sha256sum -c - | grep -qx 'a-1.2.txt: OK' || echo "not verified"
not verified
```

or skip `-c` and compare the hash itself:

```console
$ [ "$(sha256sum a.txt | cut -d' ' -f1)" = "$h" ] && echo match
match
```

## A malformed line is skipped, and the run still passes

If one line of a sums file is broken (a cut-off hash, a hand edit), it's
reported as improperly formatted and skipped. If the other lines pass, the
exit status is 0 — even when the skipped file has changed:

```console
$ printf 'tampered\n' > b.txt
$ printf '%s  a.txt\n%s  b.txt\n' "$h" e258d248fda94c63753607f7 > SUMS
$ sha256sum -c SUMS; echo "exit $?"
a.txt: OK
sha256sum: WARNING: 1 line is improperly formatted
exit 0
$ sha256sum -c --strict SUMS; echo "exit $?"
a.txt: OK
sha256sum: WARNING: 1 line is improperly formatted
exit 1
```

With the full hash, `b.txt` is checked and fails as it should:

```console
$ printf '%s  a.txt\n%s  b.txt\n' "$h" e258d248fda94c63753607f7c4494ee0fcbe92f1a76bfdac795c9d84101eb317 | sha256sum -c -; echo "exit $?"
a.txt: OK
b.txt: FAILED
sha256sum: WARNING: 1 computed checksum did NOT match
exit 1
```

Things it did accept here: one space instead of two between hash and name,
`hash *name`, upper-case hex, and the BSD `SHA256 (name) = hash` lines from
`--tag`.

## CRLF sums file: "No such file or directory"

A sums file with Windows line endings fails for a file that is right there.
This `sha256sum` at least shows the carriage return as `\r` in the message:

```console
$ printf '%s  a.txt\r\n' "$h" > CRLF.txt
$ sha256sum -c CRLF.txt; echo "exit $?"
sha256sum:  a.txt\r: No such file or directory
exit 1
$ tr -d '\r' < CRLF.txt | sha256sum -c -; echo "exit $?"
a.txt: OK
exit 0
```

## The `shasum` fallback behaves differently

`/usr/bin/shasum` (a Perl script, version 6.02 here) is the other option,
and the one to use where `/sbin` isn't on `PATH`. Three differences I hit:

Without `-a` it's SHA-1, not SHA-256:

```console
$ shasum a.txt
f572d396fae9206628714fb2ce00f72e94f2258f  a.txt
$ shasum -a 256 a.txt
5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03  a.txt
```

It reads stdin with `-c` and no file, and it fails on input that checks
nothing — where `sha256sum` exited 0:

```console
$ echo "$h  a.txt" | shasum -a 256 -c; echo "exit $?"
a.txt: OK
exit 0
$ : | shasum -a 256 -c -; echo "exit $?"
shasum: standard input: no properly formatted SHA checksum lines found
exit 1
$ echo "$h  other.iso" | shasum -a 256 -c --ignore-missing -; echo "exit $?"
shasum: standard input: no file was verified
exit 1
```

And it needs the two spaces. A one-space line is "improperly formatted",
so it's skipped — with exit 0 if another line passed, same as above:

```console
$ printf '%s  a.txt\n%s b.txt\n' "$h" e258d248fda94c63753607f7c4494ee0fcbe92f1a76bfdac795c9d84101eb317 > ONE
$ shasum -a 256 -c ONE; echo "exit $?"
a.txt: OK
shasum: WARNING: 1 line is improperly formatted
exit 0
$ sha256sum -c ONE; echo "exit $?"
a.txt: OK
b.txt: FAILED
sha256sum: WARNING: 1 computed checksum did NOT match
exit 1
```

`--strict` makes `shasum` fail on it too (exit 1).

## Checklist

- `echo "$h  file" | sha256sum -c` → usage message: write `sha256sum -c -`.
  The `-` also works with `shasum -a 256 -c`.
- `sha256sum` on macOS 26 is `/sbin/sha256sum` (md5 in GNU mode); a `PATH`
  without `/sbin` gives "command not found".
- Exit 0 from `sha256sum -c` doesn't mean a file was checked: empty input and
  `--ignore-missing` with nothing found both pass silently. Check for the
  `name: OK` line, or compare `sha256sum file | cut -d' ' -f1` with the hash.
- Add `--strict`, so a malformed line fails the run instead of being skipped.
- "No such file or directory" for a file that exists: CRLF —
  `tr -d '\r' < SUMS | sha256sum -c -`.
- With `shasum`, always pass `-a 256`, and use two spaces between hash and name.

*Tested on macOS 26.5.2 (Apple M1): `/sbin/sha256sum` ("sha256sum (Darwin)
1.0", man page dated February 13, 2024), `/usr/bin/shasum` 6.02, zsh 5.9.
All console lines replayed from this post in a fresh temporary directory with
`zsh -f -i` on a pseudo-terminal and `PATH=/usr/bin:/bin:/usr/sbin:/sbin`.
GNU coreutils' `sha256sum`, Linux, and older macOS versions were not tested,
so I can't say how GNU handles the empty-input and `--ignore-missing` cases.*
