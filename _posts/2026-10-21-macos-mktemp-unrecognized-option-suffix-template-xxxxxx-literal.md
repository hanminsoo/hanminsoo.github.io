---
title: "macOS `mktemp: unrecognized option --suffix`: a template like `report.XXXXXX.txt` creates that exact name, then \"File exists\""
description: >-
  `mktemp --suffix=.txt` fails on macOS with "mktemp: unrecognized option"
  because /usr/bin/mktemp is BSD mktemp. Moving the X's into the middle of
  the template (`report.XXXXXX.txt`) doesn't work either: in my tests it
  created a file literally named `report.XXXXXX.txt`, exited 0, and failed
  with "File exists" on the second run. For a temp file with an extension,
  make a temp directory with `mktemp -d` and put a fixed name inside it.
  Reproduced on macOS 26.
tags: [macos, shell, mktemp, bsd]
---

A script that needs a temporary file with an extension stops on a Mac:

```console
$ mkdir demo && cd demo
$ mktemp --suffix=.txt; echo "exit $?"
mktemp: unrecognized option `--suffix=.txt'
usage: mktemp [-d] [-p tmpdir] [-q] [-t prefix] [-u] template ...
       mktemp [-d] [-p tmpdir] [-q] [-u] -t prefix 
exit 1
```

`--suffix` is a GNU mktemp option (from GNU's documentation; I didn't run
GNU mktemp here). macOS ships BSD mktemp. There's no `--version`; `what`
shows the build:

```console
$ mktemp --version; echo "exit $?"
mktemp: unrecognized option `--version'
usage: mktemp [-d] [-p tmpdir] [-q] [-t prefix] [-u] template ...
       mktemp [-d] [-p tmpdir] [-q] [-u] -t prefix 
exit 1
$ what /usr/bin/mktemp
/usr/bin/mktemp:
	PROGRAM:mktemp  PROJECT:shell_cmds-329
	PROGRAM:mktemp  PROJECT:shell_cmds-329
```

## The X's in the middle: no error, no random part

The next thing to try is putting the extension after the X's:

```console
$ mktemp ./report.XXXXXX.txt; echo "exit $?"
./report.XXXXXX.txt
exit 0
$ ls
report.XXXXXX.txt
```

Nothing was replaced. The file is named `report.XXXXXX.txt`, and mktemp
exited 0, so a script carries on. The second run of the same script fails:

```console
$ mktemp ./report.XXXXXX.txt; echo "exit $?"
mktemp: mkstemp failed on ./report.XXXXXX.txt: File exists
exit 1
```

The man page says only trailing X's are replaced:

```console
$ man mktemp | col -b | grep -A1 'The trailing'
     /tmp/temp.XXXX.  The trailing ‘Xs’ are replaced with the current process
     number and/or a unique letter combination.  The number of unique file
```

A template with no X's at all behaved the same way: it created that name
and exited 0.

```console
$ mktemp ./plain.txt; echo "exit $?"
./plain.txt
exit 0
```

`-t` doesn't help either. It treats its argument as a prefix and adds its
own random part after it, so the `.txt` ends up in the middle (`-p .` puts
the file in the current folder; the random part comes out different each time, per the man page):

```console
$ mktemp -p . -t report.txt
./report.txt.kEVifr9SFB
```

## Fix: a temp directory, a fixed name inside

`mktemp -d` makes a new, empty directory with a random name (the six
characters differ on every run). Inside it, the file name can be anything:

```console
$ d=$(mktemp -d ./work.XXXXXX) && f="$d/report.txt" && : > "$f" && echo "$f"
./work.GLJA4i/report.txt
$ ls "$d"
report.txt
```

Cleanup is one `rm -r "${d:?}"` at the end:

```console
$ rm -r "${d:?}" && [ ! -e "$d" ] && echo removed
removed
```

I didn't test cleanup with `trap` here.

## Rename after `mktemp`: `mv -n` can skip silently

Another common answer is to create `report.XXXXXX` and rename it to
`report.XXXXXX.txt` (random part comes out different each time, per the man page):

```console
$ f=$(mktemp ./report.XXXXXX) && mv -n "$f" "$f.txt" && echo "$f.txt"
./report.8d6hdc.txt
```

`-n` keeps `mv` from overwriting a file that already has the `.txt` name.
But when the target existed, `mv -n` on this Mac didn't move anything and
still exited 0:

```console
$ printf 'new\n' > src; : > taken.txt
$ mv -n src taken.txt; echo "exit $?"
exit 0
$ ls src taken.txt
src
taken.txt
$ wc -c < taken.txt
       0
```

So a script that checks only `mv`'s exit status would go on writing to a
`.txt` path it didn't create. The `mktemp -d` version above doesn't have
this step.

## From Python

If the script is Python anyway, `tempfile.mkstemp` takes a suffix (the
random part comes out different each time, per the man page):

```console
$ /usr/bin/python3 -I -c 'import tempfile,os; fd,p=tempfile.mkstemp(suffix=".txt",dir="."); os.close(fd); print(os.path.basename(p))'
tmp3ngg73ki.txt
```

## Checklist

- `mktemp: unrecognized option --suffix`: `/usr/bin/mktemp` on macOS is BSD
  mktemp.
- Only trailing X's are replaced. `mktemp ./report.XXXXXX.txt` created a
  file with exactly that name and exited 0; the second run failed with
  "File exists".
- `-t name` adds the random part after the name, so the extension doesn't
  end the file name.
- Temp file with an extension: `d=$(mktemp -d ./work.XXXXXX)`, then use
  `"$d/report.txt"`.
- `mv -n` onto an existing file did nothing and exited 0 here. Don't treat
  its exit status as "renamed".

*Tested on macOS 26.5.2 (Apple silicon) with `/usr/bin/mktemp`
(`PROGRAM:mktemp  PROJECT:shell_cmds-329`), `/bin/mv` (`file_cmds-479`), and `/usr/bin/python3`
(Python 3.8.2) in zsh 5.9 (arm64-apple-darwin25.0). GNU coreutils and Linux
were not tested.*
