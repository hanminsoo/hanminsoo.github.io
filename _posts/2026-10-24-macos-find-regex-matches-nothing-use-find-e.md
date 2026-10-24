---
title: "macOS `find -regex '.*\\.(log|txt)'` finds nothing and exits 0: use `find -E` (and `-regextype: unknown primary or operator`)"
description: >-
  On macOS 26, `find . -regex '.*\.(log|txt)'` printed nothing and exited 0,
  though matching files existed. /usr/bin/find (shell_cmds-329) reads -regex as
  a basic regex unless you pass -E, so in my tests `(…|…)` and `+` didn't work
  as they do in extended regex, and the GNU-style `\(log\|txt\)` didn't match
  either. `find -E . -regex …` fixed it; -E has to come before the path.
  `-regextype` is rejected as an unknown primary. The pattern has to match the
  whole path, not just the file name.
tags: [macos, find, regex, shell, bsd]
---

A `find` that should list log and text files lists nothing, and doesn't
report an error:

```console
$ mkdir demo && cd demo
$ mkdir -p a/b && touch a/b/x.log a/b/y.txt a/z.log
$ find . -regex '.*\.(log|txt)'; echo "exit $?"
exit 0
```

There are three matching files. The `find` on this Mac reads `-regex` as a
**basic** regular expression unless you pass `-E`, and in basic regex this
alternation doesn't work, so nothing matched. Below is what I tried on
this Mac's `/usr/bin/find`, and what worked.

```console
$ what /usr/bin/find
/usr/bin/find:
	PROGRAM:find  PROJECT:shell_cmds-329
	PROGRAM:find  PROJECT:shell_cmds-329
```

## `-E` fixes it, but only before the path

```console
$ find -E . -regex '.*\.(log|txt)' | sort
./a/b/x.log
./a/b/y.txt
./a/z.log
$ find . -E -regex '.*\.(log|txt)'; echo "exit $?"
find: -E: unknown primary or operator
exit 1
```

`-E` is an option, so it goes before the starting path. After the path,
`find` reads it as an expression and fails.

The GNU way to pick the regex flavour isn't there:

```console
$ find . -regextype posix-extended -regex '.*\.(log|txt)'; echo "exit $?"
find: -regextype: unknown primary or operator
exit 1
```

## Other patterns that silently matched nothing

The default (basic) mode accepts these without an error, and returns no
results:

```console
$ find . -regex '.*\.\(log\|txt\)'; echo "exit $?"
exit 0
$ find . -regex '.*/[a-z]+\.log'; echo "exit $?"
exit 0
```

- `\(log\|txt\)` is how GNU find's default regex type writes alternation (from
  GNU's documentation; I didn't run GNU find). Here it matched nothing.
- `+` as "one or more" matched nothing without `-E`. With `-E` it works:

```console
$ find -E . -regex '.*/[a-z]+\.log' | sort
./a/b/x.log
./a/z.log
```

A group without alternation did work in basic mode:

```console
$ find . -regex '.*\.\(log\)' | sort
./a/b/x.log
./a/z.log
```

## The pattern must match the whole path

`-regex` is matched against the entire path that `find` prints, starting
with `./`, not just the file name:

```console
$ find . -regex 'x\.log'; echo "exit $?"
exit 0
$ find . -regex '.*/x\.log'
./a/b/x.log
```

## Case-insensitive, and the no-regex alternative

`-iregex` works with `-E` too:

```console
$ touch a/BIG.LOG
$ find -E . -iregex '.*\.log' | sort
./a/BIG.LOG
./a/b/x.log
./a/z.log
```

For simple suffix lists, `-name` with `-o` doesn't involve regex flavours
at all:

```console
$ find . \( -name '*.log' -o -name '*.txt' \) | sort
./a/b/x.log
./a/b/y.txt
./a/z.log
```

`-name` is case-sensitive, so `BIG.LOG` isn't listed. `-iname` includes it:

```console
$ find . \( -iname '*.log' -o -iname '*.txt' \) | sort
./a/BIG.LOG
./a/b/x.log
./a/b/y.txt
./a/z.log
```

## Checklist

- On this Mac, `find . -regex '…(a|b)…'` printed nothing and exited 0.
  Without `-E`, `/usr/bin/find` treats the pattern as basic regex.
- Use `find -E . -regex '…'`. `-E` must come **before** the path. After it,
  you get `-E: unknown primary or operator`.
- `-regextype` doesn't exist here (`unknown primary or operator`).
- In my tests, `\(…\|…\)` and `+` silently matched nothing without `-E`. A
  plain `\(…\)` group did work.
- The regex must match the whole path (`.*/name\.log`), not just the name.
- For suffix lists, `\( -name '*.log' -o -name '*.txt' \)` avoids regex
  (case-sensitive; `-iname` to ignore case).

*Tested on 2026-10-24 on macOS 26.5.2 (build 25F84, Apple silicon) with
`/usr/bin/find` (`shell_cmds-329`), every command above run in a fresh
`mktemp -d` folder under `zsh -f` (zsh 5.9, arm64-apple-darwin25.0). GNU find
and Linux were not tested.*
