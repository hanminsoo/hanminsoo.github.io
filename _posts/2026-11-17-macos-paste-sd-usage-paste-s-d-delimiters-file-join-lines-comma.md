---
title: "macOS `paste -sd,` prints \"usage: paste [-s] [-d delimiters] file ...\" — joining lines with commas, and the empty variable it leaves"
description: >-
  `cmd | paste -sd,` (join lines into "a,b,c") fails on a Mac with
  "usage: paste [-s] [-d delimiters] file ..." and exit 1: macOS paste needs a
  file operand, `-` for standard input. Inside `$(...)` the variable is just
  empty, and with another command after paste in the pipeline the script exits
  0. Also: `-d ', '` uses the two characters in turn ("web1,web2 web3"),
  `-d ''` is an error, a missing final newline glues the next output on, CRLF
  keeps the `\r`, and `tr '\n' ,` leaves a trailing comma.
tags: [macos, paste, shell, bsd]
---

The usual one-liner for turning lines into a comma-separated list, run on a
Mac:

```console
$ printf 'web1\nweb2\nweb3\n' > hosts.txt
$ cat hosts.txt | paste -sd,; echo "exit $?"
usage: paste [-s] [-d delimiters] file ...
exit 1
```

The GNU coreutils manual says `paste` reads standard input when you give it no file (I did not test this on Linux).
The `paste` that ships with macOS (`/usr/bin/paste`, text_cmds-199 here)
doesn't: its synopsis is `paste [-s] [-d list] file ...`, and the file is not
optional. The usage line is the whole error message, so it doesn't tell you
that the missing piece is a `-`.

## What it does inside a script

```console
$ cat notify.sh
#!/bin/sh
# announce which hosts we are deploying to
hosts=$(paste -sd, < hosts.txt)
echo "deploying to: $hosts"
$ sh notify.sh; echo "exit $?"
usage: paste [-s] [-d delimiters] file ...
deploying to: 
exit 0
```

The usage line goes to stderr (`2>/dev/null` or a log that only keeps stdout
hides it completely), stdout is empty, so `$hosts` is an empty string. The
script carries on and exits 0.

Whether the failure shows up in `$?` depends on where `paste` is in the
pipeline:

```console
$ paste -sd, < hosts.txt > list.txt; echo "exit $?"; wc -c < list.txt
usage: paste [-s] [-d delimiters] file ...
exit 1
       0
$ paste -sd, < hosts.txt | tr a-z A-Z; echo "exit $?"
usage: paste [-s] [-d delimiters] file ...
exit 0
```

As the last command it returns 1 (and `> list.txt` has already created an
empty file). With anything after it, the pipeline's status is the last
command's, and `tr` happily turned no input into no output: exit 0. `set -o
pipefail` brings the 1 back, but `/bin/sh` scripts often don't set it.

## Fix: give it `-`, or the file

```console
$ cat hosts.txt | paste -sd, -
web1,web2,web3
$ paste -sd, hosts.txt
web1,web2,web3
```

`-` is standard input (man paste: "If `-' is specified for one or more of the
input files, the standard input is used"). The man page's own example is
written that way: `find / -name bin -type d | paste -s -d : -`.

The script, with the failure and the empty case turned into errors:

```console
$ cat notify.sh
#!/bin/sh
# announce which hosts we are deploying to
hosts=$(paste -sd, - < hosts.txt) || exit 1
[ -n "$hosts" ] || { echo "no hosts in hosts.txt" >&2; exit 1; }
echo "deploying to: $hosts"
$ sh notify.sh; echo "exit $?"
deploying to: web1,web2,web3
exit 0
$ : > hosts.txt; sh notify.sh; echo "exit $?"
no hosts in hosts.txt
exit 1
$ printf 'web1\nweb2\nweb3\n' > hosts.txt
```

`hosts=$(...) || exit 1` works because the status of a plain assignment is the
status of its command substitution.

## `-d ', '` is two delimiters, not one

```console
$ paste -sd ', ' hosts.txt
web1,web2 web3
```

Each character of the `-d` list is a separate delimiter, used in turn: `,`
after the first line, space after the second, `,` again after the third (man
paste: "The characters in list are used circularly"). That is what makes the
man page's pairing trick work:

```console
$ printf 'a\nb\nc\nd\n' | paste -sd ',\n' -
a,b
c,d
```

but it means `paste` cannot put a multi-character separator like `", "`
between items. An empty list is an error, and `'\0'` is the way to say "no
separator":

```console
$ paste -sd '' hosts.txt; echo "exit $?"
paste: no delimiters specified
exit 1
$ paste -sd '\0' hosts.txt
web1web2web3
```

## Input edge cases

A last line without a newline: the output has no newline either, so whatever
prints next lands on the same line:

```console
$ printf 'web1\nweb2' | paste -sd, -; echo next
web1,web2next
```

Empty input gives no output at all, not even an empty line:

```console
$ printf '' | paste -sd, - | wc -c
       0
```

Windows line endings stay in the items:

```console
$ printf 'web1\r\nweb2\r\n' | paste -sd, - | od -c
0000000    w   e   b   1  \r   ,   w   e   b   2  \r  \n
0000014
```

## The other usual replacements have their own problems

```console
$ tr '\n' , < hosts.txt; echo
web1,web2,web3,
$ printf 'my host\nweb2\n' | xargs | tr ' ' ,
my,host,web2
```

`tr` also replaces the last newline, so you get a trailing comma (and no final
newline — the `echo` is there to end the line). `xargs` splits on spaces, so an
item with a space becomes two.

## A join that handles all of the above

If you need a separator longer than one character, or the input may come from
Windows, awk does it in a few lines:

```sh
# join_lines [SEP] < file -> one line, SEP between items (default ",")
# strips a trailing CR, handles a missing final newline, prints nothing for empty input
join_lines() {
  awk -v sep="${1:-,}" '
    { sub(/\r$/, ""); printf "%s%s", (NR > 1 ? sep : ""), $0 }
    END { if (NR) print "" }'
}
```

I saved it as `join.sh`:

```console
$ . ./join.sh
$ join_lines ', ' < hosts.txt
web1, web2, web3
$ printf 'web1\r\nweb2' | join_lines | od -c
0000000    w   e   b   1   ,   w   e   b   2  \n
0000012
$ printf 'my host\nweb2\n' | join_lines
my host,web2
$ printf '' | join_lines | wc -c
       0
```

`awk -v` processes backslash escapes in the value, so `join_lines '\t'` joins
with a tab; to get a literal backslash in the separator, write `'\\'`.

For a plain one-character separator, `paste -sd, -` is enough.

## Checklist

- macOS `paste` needs a file operand: `cmd | paste -sd,` prints only
  "usage: paste [-s] [-d delimiters] file ..." and exits 1. Add `-`:
  `cmd | paste -sd, -`.
- The usage line is on stderr and stdout is empty: `x=$(... | paste -sd,)`
  leaves `x` empty, and with another command after `paste` the pipeline exits 0.
  Check `|| exit 1` and `[ -n "$x" ]`.
- `-d` takes a list of single characters used in turn: `-d ', '` gives
  `web1,web2 web3`. Use awk for multi-character separators.
- `-d ''` is "no delimiters specified"; use `-d '\0'` for no separator.
- A last line without a newline gives output without a newline; empty input
  gives no output at all; `\r` from CRLF files stays in the items.
- `tr '\n' ,` leaves a trailing comma; `xargs | tr ' ' ,` splits items with
  spaces.

Related: [macOS `cut: illegal option -- -`](/2026/10/macos-cut-complement-illegal-option-drop-field/)
and [macOS `xargs: invalid option -- d`](/2026/10/macos-xargs-invalid-option-d-i-command-line-cannot-be-assembled/).

*Tested on 2026-11-17 on macOS 26.5.2 (Apple silicon) with `/usr/bin/paste`
(text_cmds-199), `/usr/bin/awk`, `tr`, `xargs`, zsh 5.9 and `/bin/sh`
(bash 3.2.57), in `env -i` with `PATH=/usr/bin:/bin:/usr/sbin:/sbin`.
Not tested: GNU coreutils `paste`, Linux, older macOS versions.*
