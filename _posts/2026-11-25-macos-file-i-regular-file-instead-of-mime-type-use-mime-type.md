---
title: "macOS `file -i` prints \"regular file\" instead of the MIME type (exit 0) — use `file --mime-type`"
description: >-
  On macOS, `file -i photo.png` prints "regular file", not "image/png", and
  exits 0. In Apple's default mode `-i` means "do not classify the contents
  of regular files"; the MIME option is `-I`. A script that sorts uploads
  with `case $(file -bi "$f") in image/*)` moves nothing and exits 0, and
  `file -bi x.txt | grep -q '^text/'` is false for every file.
  `COMMAND_MODE=legacy` flips `-i` back and makes `-I` an invalid option.
  Fix: `file -b --mime-type`, which works in both modes; take the first line
  (universal binaries print three) and treat output with a space as an error,
  because `file` exits 0 on "cannot open".
tags: [macos, file, mime-type, shell, bsd]
---

Checking a file's MIME type with `file -i` on a Mac:

```console
$ file -i /etc/hosts /bin/ls; echo "exit $?"
/etc/hosts: regular file
/bin/ls:    regular file
exit 0
```

No `text/plain`, no error, exit 0. GNU `file`'s documentation lists `-i` as
the short form of `--mime` (I did not run it on Linux for this post). On macOS (here 26.5.2, `/usr/bin/file` 5.41) `-i` means something else:

```console
$ man file | col -b | grep -E '^ +-i +If'
     -i      If the file is a regular file, do not classify its contents.
```

So `file -i` does what it says: it does not look inside the file, and every
plain file is a "regular file". The MIME option on macOS is capital `-I`
(or `--mime`). On a terminal you notice. In a script, it looks like every
file has an unknown type.

## The upload sorter that moves nothing

A small step that moves images out of an `incoming/` folder and leaves the
rest. Everything below runs in a scratch folder:

```console
$ T=$(mktemp -d)
$ cd "$T"
$ mkdir incoming
$ echo 89504e470d0a1a0a0000000d494844520000000400000004080200000026930929000000104944415478da63f8cfc000470cc47100ae930ff1385e8c110000000049454e44ae426082 | xxd -r -p > incoming/logo.png
$ echo 'meeting at 10' > incoming/notes.txt
$ echo '{"ok": true}' > incoming/data.json
$ file incoming/*
incoming/data.json: JSON data
incoming/logo.png:  PNG image data, 4 x 4, 8-bit/color RGB, non-interlaced
incoming/notes.txt: ASCII text
$ cat sort-uploads.sh
#!/bin/sh
# move images from incoming/ to images/, leave everything else where it is
mkdir -p images
for f in incoming/*; do
  type=$(file -bi "$f")
  case $type in
    image/*) mv "$f" images/ && echo "image: $f" ;;
    *)       echo "skip ($type): $f" ;;
  esac
done
$ sh sort-uploads.sh; echo "exit $?"
skip (regular file): incoming/data.json
skip (regular file): incoming/logo.png
skip (regular file): incoming/notes.txt
exit 0
$ ls images
```

(The hex line is a valid 4×4 PNG, 73 bytes, so the test needs no image file.)

`file` itself knows `logo.png` is a PNG. With `-bi` it never looks, prints
"regular file", the `case` falls through to `*)`, and `images/` stays empty.
Nothing fails, so the script exits 0.

The same happens to the common "is this a text file?" check. It is false for
a plain text file:

```console
$ file -bi incoming/notes.txt | grep -q '^text/' && echo text || echo "not text"
not text
```

Depending on which branch your script treats as "safe", that skips every file
or handles every file as binary.

## Why: Apple's `file` has two modes

The same man page explains it near the end, under legacy mode:

```console
$ man file | col -b | grep -A4 'The -d, -i, and -r options'
     The -d, -i, and -r options behave differently.  The -d option provides
     debugging information (same as -D in conformance mode).  The -i option
     displays mime type information (same as -I in conformance mode).  The -r
     option will disable the translation of unprintable characters (by
     default, this translation is already disabled in conformance mode).
```

"Conformance mode" is the default. Legacy mode is chosen with the
`COMMAND_MODE` environment variable (the same switch that changes
[`find -mtime`](/2026/11/macos-find-mtime-plus-7-not-deleting-7-day-old-files-rounded-up/)).
It is not set in this shell, and setting it to `legacy` gives `-i` the
Linux meaning, but takes `-I` away:

```console
$ echo "COMMAND_MODE=[$COMMAND_MODE]"
COMMAND_MODE=[]
$ COMMAND_MODE=legacy file -i incoming/*
incoming/data.json: application/json; charset=us-ascii
incoming/logo.png:  image/png; charset=binary
incoming/notes.txt: text/plain; charset=us-ascii
$ COMMAND_MODE=legacy file -I incoming/logo.png; echo "exit $?"
file: invalid option -- I
Usage: file [-bcCdEhikLlNnprsSvzZ0] [--extension] [--mime-encoding]
            [--mime-type] [-e <testname>] [-F <separator>]  [-f <namefile>]
            [-m <magicfiles>] [-P <parameter=value>] [--exclude-quiet]
            <file> ...
       file -C [-m <magicfiles>]
       file [--help]
Try `file --help' for more information.
exit 1
```

So neither `-i` nor `-I` works in both modes. Don't set `COMMAND_MODE` in a
script to fix this: it changes other commands too.

## What works in both modes: `--mime-type`

```console
$ file -I incoming/*
incoming/data.json: application/json; charset=us-ascii
incoming/logo.png:  image/png; charset=binary
incoming/notes.txt: text/plain; charset=us-ascii
$ file --mime-type incoming/*
incoming/data.json: application/json
incoming/logo.png:  image/png
incoming/notes.txt: text/plain
$ COMMAND_MODE=legacy file --mime-type incoming/*
incoming/data.json: application/json
incoming/logo.png:  image/png
incoming/notes.txt: text/plain
$ file -b --mime-type incoming/logo.png
image/png
```

`--mime-type` prints only the type, without `; charset=...`, which is also
what a `case` or `grep` usually wants. GNU `file` documents the same long
option, but I did not run GNU `file` for this post.

Two more things showed up when I used it in a script.

**Universal binaries print more than one line**, even with `-b`:

```console
$ file -b --mime-type /bin/ls
application/x-mach-binary
/bin/ls (for architecture x86_64):	application/x-mach-binary
/bin/ls (for architecture arm64e):	application/x-mach-binary
```

The first line is the whole file; take that one with `head -n 1`.

**`file` exits 0 when it cannot read the file.** The error text goes to
stdout in place of the type (shown in the run below:
`cannot open: Permission denied`). `--mime-type` output (type/subtype) has no space, so
treat empty output or output with a space as an error.

## The fixed sorter

```console
$ cat sort-uploads.sh
#!/bin/sh
# move images from incoming/ to images/, leave everything else where it is
mkdir -p images
for f in incoming/*; do
  type=$(file -b --mime-type "$f" | head -n 1)
  case $type in
    ''|*' '*) echo "no MIME type for $f: $type" >&2; exit 2 ;;
    image/*)  mv "$f" images/ && echo "image: $f" ;;
    *)        echo "skip ($type): $f" ;;
  esac
done
$ sh sort-uploads.sh; echo "exit $?"
skip (application/json): incoming/data.json
image: incoming/logo.png
skip (text/plain): incoming/notes.txt
exit 0
$ ls images incoming
images:
logo.png

incoming:
data.json
notes.txt
```

And a file `file` cannot read stops the script instead of being sorted as
"not an image":

```console
$ echo 89504e470d0a1a0a > incoming/locked.png; chmod 000 incoming/locked.png
$ sh sort-uploads.sh; echo "exit $?"
skip (application/json): incoming/data.json
no MIME type for incoming/locked.png: cannot open: Permission denied
exit 2
```

Clean up the scratch folder:

```console
$ cd /
$ chmod 600 "$T/incoming/locked.png"
$ rm -r "${T:?}"
```

## Checklist

- On macOS (default mode), `file -i` means "do not classify regular files":
  every plain file is `regular file`, exit 0. It is not the MIME option.
- The macOS MIME option is `-I` / `--mime`, but `-I` is an invalid option
  when `COMMAND_MODE=legacy` (where `-i` is the MIME option instead).
- Use `file -b --mime-type "$f"`: it gave the same output in both modes here.
- Take the first line (`| head -n 1`); universal binaries add one line per
  architecture.
- `file` exits 0 on "cannot open"; reject empty output or output containing
  a space before you act on the type.
- A `case` that falls through to a default branch hides all of this. Print
  the type you got in that branch (`skip ($type)`), so "regular file" shows up.

Related: [macOS `find -mtime +7` and `COMMAND_MODE`](/2026/11/macos-find-mtime-plus-7-not-deleting-7-day-old-files-rounded-up/)
and [macOS `grep: invalid option -- P`](/2026/10/macos-grep-invalid-option-p-lookahead-perl-ne/).

*Tested on 2026-11-25 on macOS 26.5.2 (Apple silicon, APFS) with
`/usr/bin/file` (file-5.41, Apple file-106), `/usr/bin/xxd`, zsh 5.9 and
`/bin/sh` (bash 3.2.57), in `env -i` with `PATH=/usr/bin:/bin:/usr/sbin:/sbin`.
Not tested: GNU `file` on Linux, Homebrew's `file`, older macOS versions.*
