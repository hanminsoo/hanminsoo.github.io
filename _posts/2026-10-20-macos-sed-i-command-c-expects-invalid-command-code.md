---
title: "macOS `sed -i`: \"invalid command code\", \"command c expects \\ followed by text\", and the stray `file-e` backup"
description: >-
  On macOS, `sed -i 's/a/b/' file` fails with an error that depends on the
  file's first letter: "invalid command code f", "command c expects \ followed
  by text", "extra characters at the end of h command", and so on. BSD sed's
  bare -i takes the next argument as the backup suffix, so the script is used as the
  suffix and the file name is parsed as the script. `sed -i -e ...` "works" but
  leaves a `file-e` backup, and `sed -i -E ...` silently stops using extended
  regex and changes nothing. Fixes: `sed -i '' ...`, `sed -i.bak ... && rm`,
  or `perl -i -pe`. Reproduced with /usr/bin/sed on macOS 26.
tags: [macos, sed, shell, bsd]
---

A one-liner that edits a file in place works on Linux and fails on a Mac:

```console
$ mkdir demo && cd demo
$ printf 'port=8080\n' > conf.txt
$ sed -i 's/8080/9090/' conf.txt; echo "exit $?"
sed: 1: "conf.txt
": command c expects \ followed by text
exit 1
```

The message doesn't mention `-i`, and it changes with the file name, which
makes it hard to search for. Below: why that happens, two variants that
*don't* fail but do the wrong thing (a `conf.txt-e` file, and a silent
no-op with `-E`), and three fixes.

## Why: BSD `-i` takes the next argument as the backup suffix

macOS ships BSD sed. Its usage line shows that `-i` takes an argument:

```console
$ sed --version
sed: illegal option -- -
usage: sed script [-EHalnru] [-i extension] [file ...]
	sed [-EHalnu] [-i extension] [-e script] ... [-f script_file] ... [file ...]
```

So in `sed -i 's/8080/9090/' conf.txt`, `s/8080/9090/` becomes the backup
suffix, and `conf.txt` is the only argument left, so sed takes it as the
**script**. The script `conf.txt` starts with `c`, which is sed's "change"
command, hence "command c expects \ followed by text". With other file names
you get other errors:

```console
$ for f in file.txt app.conf hosts settings.txt; do printf 'port=8080\n' > $f; sed -i 's/8080/9090/' $f; done
sed: 1: "file.txt
": invalid command code f
sed: 1: "app.conf
": command a expects \ followed by text
sed: 1: "hosts
": extra characters at the end of h command
sed: 1: "settings.txt
": unterminated substitute pattern
$ rm file.txt app.conf hosts settings.txt
```

`f` is not a sed command, so that's "invalid command code f". `a` (append),
`h` (hold), and `s` (substitute) are commands, so you get a complaint about
what follows them instead. All four are the same mistake.

## `sed -i -e`: works, and leaves `conf.txt-e`

A common "fix" is adding `-e` in front of the script. It runs and exits 0:

```console
$ sed -i -e 's/8080/9090/' conf.txt; echo "exit $?"
exit 0
$ ls -1
conf.txt
conf.txt-e
$ cat conf.txt conf.txt-e
port=9090
port=8080
```

The edit happened, but `-e` was taken as the backup suffix, so the original
was saved as `conf.txt-e`. Run that over a folder of config files and you
get one `-e` file next to each of them. The edit still happened because,
once `-i` had taken `-e`, the next argument `s/8080/9090/` was the first
non-option, and sed used it as the script.

## `sed -i -E`: exit 0, nothing changed

The same thing with `-E` (extended regex) is worse, because it fails
silently:

```console
$ rm conf.txt-e; printf 'port=8080\n' > conf.txt
$ sed -i -E 's/[0-9]+/9090/' conf.txt; echo "exit $?"
exit 0
$ ls -1; cat conf.txt
conf.txt
conf.txt-E
port=8080
```

`-E` became the backup suffix, so the script ran as a **basic** regex. In
basic regex, `+` is a literal plus sign, so `[0-9]+` matched nothing, and
sed wrote the file back unchanged and exited 0. The only sign is the
`conf.txt-E` file.

## Fixes

**macOS only: give `-i` an explicit empty suffix**, as a separate argument:

```console
$ rm conf.txt-E
$ sed -i '' -E 's/[0-9]+/9090/' conf.txt; echo "exit $?"
exit 0
$ ls -1; cat conf.txt
conf.txt
port=9090
```

**Attach a suffix to `-i` and delete the backup afterwards:**

```console
$ printf 'port=8080\n' > conf.txt
$ sed -i.bak -E 's/[0-9]+/9090/' conf.txt && rm conf.txt.bak; echo "exit $?"
exit 0
$ ls -1; cat conf.txt
conf.txt
port=9090
```

GNU sed's documentation describes `-i` as taking an *optional* suffix that
must be attached (`-i.bak`), which is why `sed -i 's/…/'` works there. By
that reading, `-i.bak` means the same thing in both seds, and `-i ''`
would break on Linux. I didn't have GNU sed here, so I haven't tested
either claim.

**Use perl**, which ships with macOS and has one `-i` behaviour (I tested it on macOS only):

```console
$ printf 'port=8080\n' > conf.txt
$ perl -i -pe 's/\d+/9090/' conf.txt; echo "exit $?"
exit 0
$ ls -1; cat conf.txt
conf.txt
port=9090
```

## Checklist

- macOS `/usr/bin/sed` is BSD sed. A bare `-i` (nothing attached, unlike
  `-i.bak`) takes the next argument as the backup suffix.
- `sed -i 'script' file`: the script becomes the suffix and the file name
  becomes the script. The error depends on the name's first letter:
  `invalid command code f`, `command c expects \ followed by text`,
  `extra characters at the end of h command`,
  `unterminated substitute pattern`...
- `sed -i -e 'script' file` works but leaves `file-e`.
- `sed -i -E 'script' file` exits 0 without `-E`. An extended regex then
  silently matches nothing. Look for a `file-E`.
- macOS: `sed -i '' ...`. On both macOS and Linux: `sed -i.bak ... && rm
  file.bak` (GNU side not tested here), or `perl -i -pe`.
- After an in-place edit, look at the file. An exit code of 0 doesn't mean
  it changed.

*Tested on macOS 26.5.2 (Apple silicon) with `/usr/bin/sed` (`what
/usr/bin/sed`: `PROGRAM:sed  PROJECT:text_cmds-199`; it has no `--version`),
the system perl (`This is perl 5, version 34, subversion 1 (v5.34.1) built
for darwin-thread-multi-2level`), and zsh 5.9 (arm64-apple-darwin25.0).
GNU sed and Linux were not tested.*
