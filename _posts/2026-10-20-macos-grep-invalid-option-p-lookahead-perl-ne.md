---
title: "macOS `grep: invalid option -- P`: what `grep -E` already handles, the `[\\d]` trap, and `perl -ne` for lookahead (with grep's exit status)"
description: >-
  `grep -P` fails on macOS with "grep: invalid option -- P" (exit 2), because
  /usr/bin/grep is BSD grep 2.6.0-FreeBSD. On this grep, `-E` accepts `\d`,
  `\s`, `\w`, `\b` and lazy `.+?`, but inside brackets `[\d]` means
  "backslash or d", not a digit. Lookahead and lookbehind fail with
  "repetition-operator operand invalid". For those, use `perl -ne`, but perl
  exits 0 even when nothing matches, so add `END { exit !$m }` to get grep's
  exit status back. Reproduced on macOS 26.
tags: [macos, grep, perl, shell, regex]
---

A grep line copied from a Linux script or an answer online fails on a Mac:

```console
$ mkdir demo && cd demo
$ printf '%s\n' 'id=42 user=dan' 'id=x7 user=ada' 'port 8080 open' 'done' > log.txt
$ grep -P 'id=\d+' log.txt; echo "exit $?"
grep: invalid option -- P
usage: grep [-abcdDEFGHhIiJLlMmnOopqRSsUVvwXxZz] [-A num] [-B num] [-C[num]]
	[-e pattern] [-f file] [--binary-files=value] [--color=when]
	[--context[=num]] [--directories=action] [--label] [--line-buffered]
	[--null] [pattern] [file ...]
exit 2
$ grep --version
grep (BSD grep, GNU compatible) 2.6.0-FreeBSD
```

`-P` (Perl-compatible regex) is a GNU grep option, and the grep that ships
with macOS is BSD grep. Below is what I checked on this Mac: which `-P`
patterns work as they are with `-E`, one bracket trap, and how to handle
lookahead with `perl`, keeping grep's exit status.

## Many `-P` patterns work with `-E` here

```console
$ grep -E 'id=\d+' log.txt; echo "exit $?"
id=42 user=dan
exit 0
$ echo 'id=ddd' | grep -E 'id=\d+'; echo "exit $?"
exit 1
$ grep -E 'port\s\w+' log.txt
port 8080 open
$ echo 'cat concat' | grep -oE '\bcat\b'
cat
$ echo '<a><b>' | grep -oE '<.+?>'
<a>
<b>
```

On this grep, `\d` is a digit (it doesn't match a literal `ddd`), `\s`,
`\w` and `\b` work, and `.+?` is lazy: it stops at the first `>`, giving
two matches rather than one `<a><b>`. So for a lot of `-P` lines, switching
to `-E` is enough.

These escapes are extensions, not POSIX ERE. I didn't test them on
GNU grep or other systems, so if the script has to run elsewhere, use the
plain forms: `[0-9]`, `[[:space:]]`, `[[:alnum:]_]`.

## Trap: `[\d]` is not a digit class

Inside brackets, the backslash loses its meaning. `[\d]` is "a backslash
or the letter d":

```console
$ printf '%s\n' 'a5' 'ad' 'a\' | grep -E 'a[\d]'
ad
a\
```

`a5` doesn't match, but `ad` and `a\` do, and there's no error. A pattern like
`[\d.]+` for version numbers goes wrong the same way. (`[\w-]` did match
`axb`, `a_b` and `a-b` in my test, so `\w` behaves differently from `\d`
here; I didn't dig into why.) Inside brackets, `[0-9.]` and `[[:alnum:]_-]`
are the safe spellings.

## Lookahead and lookbehind: an error

```console
$ grep -E 'user=(?!dan)' log.txt; echo "exit $?"
grep: repetition-operator operand invalid
exit 2
$ grep -E '(?<=user=)a' log.txt; echo "exit $?"
grep: repetition-operator operand invalid
exit 2
```

The message is about `?` having nothing to repeat. It doesn't say that
lookaround isn't supported. For these, use perl, which ships with macOS:

```console
$ perl -v | sed -n 2p
This is perl 5, version 34, subversion 1 (v5.34.1) built for darwin-thread-multi-2level
$ perl -ne 'print if /user=(?!dan)/' log.txt
id=x7 user=ada
```

`-n` loops over input lines and `print if /re/` prints matching lines, like
`grep 're'`. To print only the matched part, like `grep -o`, including
lookbehind:

```console
$ perl -nle 'print $& while /(?<=user=)\w+/g' log.txt
dan
ada
```

For a capture group: `perl -ne 'print "$1\n" if /id=(\d+)/'`.

```console
$ perl -ne 'print "$1\n" if /id=(\d+)/' log.txt
42
```

## Trap: `perl -ne` exits 0 when nothing matches

grep exits 1 when there's no match, and scripts depend on that
(`if grep -q ...`). perl doesn't:

```console
$ perl -ne 'print if /nomatch/' log.txt; echo "exit $?"
exit 0
```

Remember whether anything matched, and set the exit status at the end:

```console
$ perl -ne '$m=1, print if /nomatch/; END { exit !$m }' log.txt; echo "exit $?"
exit 1
$ perl -ne '$m=1, print if /id=\d+/; END { exit !$m }' log.txt; echo "exit $?"
id=42 user=dan
exit 0
```

With several files, grep puts the file name in front of each line. In
perl, the current file name is `$ARGV`:

```console
$ cp log.txt log2.txt
$ perl -ne 'print "$ARGV:$_" if /user=(?!dan)/' log.txt log2.txt
log.txt:id=x7 user=ada
log2.txt:id=x7 user=ada
```

## Checklist

- `grep: invalid option -- P` (exit 2): macOS grep is BSD grep 2.6.0-FreeBSD,
  with no PCRE.
- Try `-E` first. On this grep, `\d \s \w \b` and lazy `.+?` work
  (not tested elsewhere; use `[0-9]`, `[[:space:]]` for portability).
- `[\d]` inside brackets means backslash or `d`, silently. Write `[0-9]`.
- Lookahead or lookbehind in `-E`: `repetition-operator operand invalid`.
  Use `perl -ne 'print if /re/'`.
- `grep -o` with perl: `perl -nle 'print $& while /re/g'`.
- `perl -ne` exits 0 even with no match. Add `$m=1, ... END { exit !$m }`
  if the script checks the exit status.

*Tested on macOS 26.5.2 (Apple silicon) with `/usr/bin/grep` (`grep (BSD
grep, GNU compatible) 2.6.0-FreeBSD`) and the system perl (v5.34.1), in zsh 5.9
(arm64-apple-darwin25.0). GNU grep and Linux were not tested.*
