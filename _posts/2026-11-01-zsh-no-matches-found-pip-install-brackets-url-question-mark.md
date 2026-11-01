---
title: "zsh: no matches found: requests[socks] (pip install) or a URL with `?`: quote it"
description: >-
  On macOS, zsh is the default shell, and `pip install requests[socks]` or
  `curl https://host/page?id=1` stops with "zsh: no matches found" and exit 1:
  `[...]`, `?` and `*` are filename patterns, and zsh refuses to run a command
  whose pattern matches no file. Quote the argument (or use `noglob`). In a
  zsh script the failure is easy to miss: the external command just doesn't
  run, the script carries on and can still exit 0 (a curl that never ran left
  yesterday's file in place). With a builtin such as `echo` or `[`, the whole
  script stops instead, which is also where `[ "$a" == x ]` gives "= not
  found". `setopt nonomatch` and bash pass the text through only as long as no
  file happens to match.
tags: [macos, zsh, shell, glob, pip, curl]
---

You copy an install line from a README into Terminal on a Mac and get this,
with nothing installed:

```console
$ T=$(mktemp -d) && cd "$T"
$ python3 -m pip install requests[socks]; echo "exit $?"
zsh: no matches found: requests[socks]
exit 1
```

Same thing with a URL that has a query string:

```console
$ curl -s http://127.0.0.1:8765/page.html?v=2; echo "exit $?"
zsh: no matches found: http://127.0.0.1:8765/page.html?v=2
exit 1
```

The message comes from zsh, not from pip or curl. Neither program was
started. (zsh has been the default shell on macOS since Catalina; a README
written for bash may leave out the quotes, because bash runs the same line.)

The examples below write small files (`page.html`, `fetch.sh`, `a.zsh` …)
into the current folder and overwrite files with those names, so run them in
a new empty folder, as the first line above does: `T=$(mktemp -d) && cd "$T"`.
The last section removes it.

## Why: `[ ]`, `?` and `*` are filename patterns

Before running a command, the shell expands unquoted patterns into the file
names they match. `requests[socks]` is a pattern: "`requests` followed by one
of the characters s, o, c, k". `page.html?v=2` has `?`, "any one character".
When nothing in the folder matches, zsh's default (the `NOMATCH` option) is to
print `no matches found` and not run the command. bash and `/bin/sh` (bash
3.2 in sh mode on macOS) instead pass the pattern through unchanged, which is
why the README line works there:

```console
$ bash -c 'printf "<%s>\n" requests[socks] http://x/a?id=1'
<requests[socks]>
<http://x/a?id=1>
$ /bin/sh -c 'printf "<%s>\n" requests[socks] http://x/a?id=1'
<requests[socks]>
<http://x/a?id=1>
```

`printf '<%s>\n'` prints each argument the command receives in angle
brackets. I use it below to show exactly what a program would get.

## The fix: quote the argument

Single or double quotes turn the pattern back into plain text:

```console
$ printf '<%s>\n' "requests[socks]" 'http://x/a?id=1'
<requests[socks]>
<http://x/a?id=1>
$ noglob printf '<%s>\n' requests[socks]
<requests[socks]>
$ printf '<%s>\n' requests\[socks\]
<requests[socks]>
```

So the install line is `python3 -m pip install "requests[socks]"`, and the URL
is `curl "https://…?id=1"`. `noglob` (zsh only) turns off patterns for one
command; a backslash escapes one character. Quotes are what I'd use, because
they also work in bash and sh, and a URL may contain `&`, which quotes protect
too.

To check that pip now receives `requests[socks]` as one argument, without
downloading anything, `--no-index` makes it fail at the lookup step and name
what it was asked for:

```console
$ python3 -m pip install --no-index "requests[socks]"
Defaulting to user installation because normal site-packages is not writeable
ERROR: Could not find a version that satisfies the requirement requests[socks] (from versions: none)
ERROR: No matching distribution found for requests[socks]
```

That's pip's own error now, not zsh's, and it shows the `[socks]` arrived.
(This only shows what pip was given; I didn't install anything.)

## In a script, the command is skipped and the script keeps going

At the prompt, the error is in your face. In a zsh script it isn't. Here's a
script that downloads a page and then uses it; `page.html` is still there from
an earlier run, holding `old`, and the server now serves `new`:

```console
$ mkdir site && echo new > site/page.html && echo old > page.html
$ SRV=$(python3 -m http.server 8765 --bind 127.0.0.1 --directory site >/dev/null 2>&1 & echo $! ); sleep 1
$ printf '#!/bin/zsh\ncurl -fsS -o page.html http://127.0.0.1:8765/page.html?v=2\necho "fetched: $(cat page.html)"\n' > fetch.sh && chmod +x fetch.sh
$ cat fetch.sh
#!/bin/zsh
curl -fsS -o page.html http://127.0.0.1:8765/page.html?v=2
echo "fetched: $(cat page.html)"
$ ./fetch.sh; echo "exit $?"
./fetch.sh:2: no matches found: http://127.0.0.1:8765/page.html?v=2
fetched: old
exit 0
```

(The space in `$! )` matters if you paste this into an interactive zsh:
there, `$!)` is taken as history expansion and the line fails with
`event not found: )`.)

curl never ran, the next line read the stale file, and the script exited 0
because its last command succeeded. In a log, that one line on stderr is the
only trace. With the URL quoted:

```console
$ printf '#!/bin/zsh\ncurl -fsS -o page.html "http://127.0.0.1:8765/page.html?v=2"\necho "fetched: $(cat page.html)"\n' > fetch.sh
$ ./fetch.sh; echo "exit $?"
fetched: new
exit 0
$ kill "$SRV"
```

The skipped command itself does get status 1, so `set -e` or `|| exit 1`
stops the script there. (These `rm -f old?.log` lines are meant for the empty
folder: there is no `old?.log` there, which is the point. In a folder that has
`old1.log`, they would delete it.)

```console
$ zsh -f -c 'rm -f old?.log; echo "rm exit $?"'
zsh:1: no matches found: old?.log
rm exit 1
$ zsh -f -c 'set -e; rm -f old?.log; echo "still running"'; echo "exit $?"
zsh:1: no matches found: old?.log
exit 1
```

### With a builtin, the whole script stops

What happens next depends on the command. In my tests, when the command with
the unmatched pattern was an external program (`rm`, `curl` above, `ls`,
`python3`, `/bin/echo`, `command echo`), the script went on to the next line.
When it was a zsh builtin (`echo`, `print`, `printf`), the script ended right
there with exit 1:

```console
$ printf 'echo start\nrm -f old?.log\necho next line\n' > a.zsh; zsh -f a.zsh; echo "exit $?"
start
a.zsh:2: no matches found: old?.log
next line
exit 0
$ printf 'echo start\necho old?.log\necho next line\n' > b.zsh; zsh -f b.zsh; echo "exit $?"
start
b.zsh:2: no matches found: old?.log
exit 1
$ for c in ls /bin/echo "command echo" "python3 -c 1" print printf; do printf '%s old?.log\necho next line\n' "$c" > c.zsh; echo "== $c"; zsh -f c.zsh; echo "exit $?"; done
== ls
c.zsh:1: no matches found: old?.log
next line
exit 0
== /bin/echo
c.zsh:1: no matches found: old?.log
next line
exit 0
== command echo
c.zsh:1: no matches found: old?.log
next line
exit 0
== python3 -c 1
c.zsh:1: no matches found: old?.log
next line
exit 0
== print
c.zsh:1: no matches found: old?.log
exit 1
== printf
c.zsh:1: no matches found: old?.log
exit 1
```

I haven't looked into why; this is what zsh 5.9 did here, both with
`zsh -f` and with `#!/bin/zsh`. Either way, don't rely on it: quote the
argument, or let `set -e` stop the script.

## The same family: `[ "$a" == x ]` gives "= not found"

A related zsh expansion catches `==` in the `[` test that bash accepts. A
word that starts with `=` is replaced by the path of the command with that
name (the `EQUALS` option), so `==` asks for a command called `=`:

```console
$ zsh -f -c 'a=x; if [ "$a" == x ]; then echo same; fi; echo "exit $?"'; echo "outer exit $?"
zsh:1: = not found
outer exit 1
$ zsh -f -c 'a=x; if [ "$a" = x ]; then echo same; fi'
same
$ zsh -f -c 'a=x; if [[ $a == x ]]; then echo same; fi'
same
$ bash -c 'a=x; if [ "$a" == x ]; then echo same; fi'
same
```

`[` is a builtin, so as above, the rest of the script didn't run (`exit $?`
never printed). Use `=` inside `[ ]`, or `==` inside `[[ ]]`.

## `setopt nonomatch` and bash: fine until a file matches

A common tip is to put `setopt nonomatch` in `~/.zshrc`, so zsh behaves like
bash and passes an unmatched pattern through. That hides the error, but the
argument is still a pattern. The day a file in the current folder matches, the
program gets the file name instead:

```console
$ zsh -f -c 'setopt nonomatch; printf "<%s>\n" requests[socks]; touch requestss; printf "<%s>\n" requests[socks]; rm requestss'
<requests[socks]>
<requestss>
$ bash -c 'printf "<%s>\n" requests[socks]; touch requestso; printf "<%s>\n" requests[socks]; rm requestso'
<requests[socks]>
<requestso>
```

No error either time; `pip install requests[socks]` in that folder would ask
pip for something called `requestss`. (Each line removes only the one file it
just created.) bash does the same. Quoting is the fix
in both shells; `nonomatch` only changes which way it goes wrong.

## curl has its own brackets

Quoting gets the URL past the shell, but curl itself also treats `[1-2]` and
`{a,b}` in a URL as ranges and fetches each one. Here the quoted URL was
fetched twice (the page holds `new`):

```console
$ SRV=$(python3 -m http.server 8765 --bind 127.0.0.1 --directory site >/dev/null 2>&1 & echo $! ); sleep 1
$ curl -fsS "http://127.0.0.1:8765/page.html?n=[1-2]"
new
new
$ curl -fsS -g "http://127.0.0.1:8765/page.html?n=[1-2]"
new
$ kill "$SRV"
```

`-g` (`--globoff`) turns that off.

## Clean up

Leave the test folder and remove it. `${T:?}` stops the line if `T` is
empty instead of running `rm -rf` on nothing in particular:

```console
$ cd ~ && rm -rf "${T:?}"; ls -d "$T"
ls: /var/folders/xx/xxxxxxxx/T/tmp.XXXXXXXXXX: No such file or directory
```

(The folder name is random; I replaced the random parts with `x`/`X`.)

## Checklist

- `zsh: no matches found: …` means zsh read `[ ]`, `?` or `*` in an
  unquoted word as a file pattern, found no file, and didn't run the command.
  Exit 1.
- Quote such arguments: `pip install "requests[socks]"`, `curl "https://…?a=1"`.
  `noglob cmd …` or a backslash also work in zsh.
- In a zsh script, an external command with an unmatched pattern is skipped
  and the script continues (it can still end with exit 0); with a builtin
  such as `echo` or `[`, the script stops with exit 1. Use `set -e` or
  `|| exit 1`, and quote.
- `[ "$a" == x ]` in zsh: "= not found". Use `[ "$a" = x ]` or `[[ $a == x ]]`.
- `setopt nonomatch` (and bash's default) passes the text through only while
  no file matches; then the argument silently becomes a file name.
- curl expands `[1-2]` and `{a,b}` in URLs itself; add `-g` if they're
  literal.

*Tested on 2026-11-01 on macOS 26.5.2 (build 25F84, Apple silicon) with zsh
5.9 (`zsh -f`; the interactive lines replayed in `zsh -f -i`, and
`#!/bin/zsh` for fetch.sh), bash 3.2.57 (also as `/bin/sh`), `/usr/bin/python3`
3.8.2 with pip 25.0.1 (`python3 -m pip --version`), curl 8.7.1, and `python3 -m http.server` on 127.0.0.1. Not tested:
zsh with oh-my-zsh or other plugins that change typed or pasted URLs, newer
bash, Linux.*
