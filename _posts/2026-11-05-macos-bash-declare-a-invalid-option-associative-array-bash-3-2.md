---
title: "macOS bash: \"declare: -A: invalid option\" — the script keeps going and keys usually land in slot 0 (bash 3.2)"
description: >-
  On macOS, /bin/bash is bash 3.2.57, so a script that uses an associative
  array prints "declare: -A: invalid option" and then carries on: the array
  is created as an ordinary indexed array, each key such as `apple` or
  `my-key` is usually read as arithmetic (0), every assignment overwrites slot 0, and
  `${color[apple]}` returns the last value written, with exit 0. `mapfile`,
  `readarray`, `${x,,}`, `${x^^}`, `${a[-1]}`, `shopt -s globstar`, `|&` and
  `declare -n` fail the same way. Of the two I ran through `bash -n`, only `|&` is caught. Put a
  version check at the top so the script stops, or write it for 3.2: a `case`
  lookup, a `while read` loop, `tr`, `${a[@]: -1}`, `find`, `2>&1 |`.
tags: [macos, bash, shell, associative-array]
---

A small lookup script that uses an associative array, run with the Mac's own bash:

```console
$ T=$(mktemp -d) && cd "$T"
$ cat > assoc.sh <<'EOF'
#!/usr/bin/env bash
declare -A color
color[apple]=red
color[sky]=blue
echo "apple is ${color[apple]}"
echo "sky is ${color[sky]}"
EOF
$ bash assoc.sh; echo "exit $?"
assoc.sh: line 2: declare: -A: invalid option
declare: usage: declare [-afFirtx] [-p] [name[=value] ...]
apple is blue
sky is blue
exit 0
```

There is an error message, but the script doesn't stop there, it prints a
wrong answer (`apple is blue`), and the exit status is 0. If the output goes
to a log or another program, nothing looks broken.

The examples write small files (`assoc.sh`, `list.txt`, …) into the current
folder, so run them in an empty folder like the `mktemp -d` one above.

## Why: /bin/bash on macOS is 3.2

```console
$ bash --version | head -n 1
GNU bash, version 3.2.57(1)-release (arm64-apple-darwin25)
$ which -a bash
/bin/bash
```

Associative arrays (`declare -A`) are a bash 4 feature (per the bash
changelog; I had no bash 4 here to run). bash 3.2 doesn't know the `-A` option, so `declare` fails (status 2) and the script continues with
the next line. `color[apple]=red` is still valid bash 3.2, though: it creates
an ordinary indexed array, and the part in brackets is an *arithmetic*
expression. `apple` is the name of an unset variable, which counts as 0.
So both lines write to `color[0]`:

```console
$ bash -c 'declare -A c; c[apple]=red; c[sky]=blue; declare -p c' 2>/dev/null
declare -a c='([0]="blue")'
```

A key with a dash is arithmetic too (`my - key` = 0 − 0):

```console
$ bash -c 'declare -A c; c[my-key]=1; c[other]=2; declare -p c' 2>/dev/null
declare -a c='([0]="2")'
```

And if a variable with the key's name happens to hold a number, the value
goes to that slot instead:

```console
$ bash -c 'declare -A c; sky=3; c[sky]=blue; declare -p c' 2>/dev/null
declare -a c='([3]="blue")'
```

Only a key that isn't a valid expression stops the script:

```console
$ bash -c 'declare -A c; c["light blue"]=1; echo after'; echo "exit $?"
bash: line 0: declare: -A: invalid option
declare: usage: declare [-afFirtx] [-p] [name[=value] ...]
bash: light blue: syntax error in expression (error token is "blue")
exit 1
```

`local -A` inside a function behaves the same way:

```console
$ bash -c 'f() { local -A m; m[a]=1; echo "${m[a]}"; }; f; echo "exit $?"'
bash: line 0: local: -A: invalid option
local: usage: local name[=value] ...
1
exit 0
```

## The rest of the newer-bash family

Scripts written for a newer bash often use a few more features that 3.2
doesn't have. In 3.2:

```console
$ bash -c 'mapfile -t lines < /etc/shells; echo "${#lines[@]} lines"'; echo "exit $?"
bash: mapfile: command not found
0 lines
exit 0
$ bash -c 'readarray -t lines < /etc/shells'; echo "exit $?"
bash: readarray: command not found
exit 127
$ bash -c 'n=Hello; echo "${n,,}"'; echo "exit $?"
bash: ${n,,}: bad substitution
exit 1
$ bash -c 'n=Hello; echo "${n^^}"'; echo "exit $?"
bash: ${n^^}: bad substitution
exit 1
$ bash -c 'a=(x y z); echo "[${a[-1]}]"'; echo "exit $?"
bash: a: bad array subscript
[]
exit 0
$ bash -c 'shopt -s globstar'; echo "exit $?"
bash: line 0: shopt: globstar: invalid shell option name
exit 1
$ bash -c 'echo hi |& cat'; echo "exit $?"
bash: -c: line 0: syntax error near unexpected token `&'
bash: -c: line 0: `echo hi |& cat'
exit 2
$ bash -c 'declare -n r=x'; echo "exit $?"
bash: line 0: declare: -n: invalid option
declare: usage: declare [-afFirtx] [-p] [name[=value] ...]
exit 2
```

Note the ones that keep going: `mapfile` is "command not found" and the
script continues with an empty array (`0 lines`, exit 0), and `${a[-1]}`
prints an error and expands to nothing. `${n,,}` ends the `bash -c` here
with exit 1.

`bash -n` (syntax check only) is no help for the ones I tried (`declare -A`), because they
are failures at run time, not syntax errors. Only `|&` is caught:

```console
$ bash -n assoc.sh; echo "exit $?"
exit 0
$ echo 'echo hi |& cat' > pipe.sh; bash -n pipe.sh; echo "exit $?"
pipe.sh: line 1: syntax error near unexpected token `&'
pipe.sh: line 1: `echo hi |& cat'
exit 2
```

`set -e` does stop at the failing `declare` (status 2) or at `mapfile`
(127), so a script with `set -e` fails loudly instead of printing a wrong
answer:

```console
$ bash -c 'set -e; declare -A c; echo after'; echo "exit $?"
bash: line 0: declare: -A: invalid option
declare: usage: declare [-afFirtx] [-p] [name[=value] ...]
exit 2
$ bash -c 'set -e; mapfile -t l < /etc/shells; echo after'; echo "exit $?"
bash: mapfile: command not found
exit 127
```

## Fix 1: stop at the top if bash is too old

If the script really needs a newer bash, say so on the first lines, before
anything else runs. The check below asks for 4, enough for `declare -A`,
`mapfile` and `${x,,}`; the bash NEWS file lists `declare -n` and negative
subscripts as later 4.x additions, so raise the number if you use those
(not checked here):

```console
$ cat > needs4.sh <<'EOF'
#!/usr/bin/env bash
if [ -z "${BASH_VERSINFO:-}" ] || [ "${BASH_VERSINFO[0]}" -lt 4 ]; then
  echo "${0##*/}: needs bash 4 or newer, this is ${BASH_VERSION:-not bash}" >&2
  exit 1
fi
declare -A color=([apple]=red [sky]=blue)
echo "apple is ${color[apple]}"
EOF
$ bash needs4.sh; echo "exit $?"
needs4.sh: needs bash 4 or newer, this is 3.2.57(1)-release
exit 1
$ sh needs4.sh; echo "exit $?"
needs4.sh: needs bash 4 or newer, this is 3.2.57(1)-release
exit 1
$ zsh -f needs4.sh; echo "exit $?"
needs4.sh: needs bash 4 or newer, this is not bash
exit 1
```

(`sh` on macOS is bash 3.2 too. zsh has no `BASH_VERSINFO`, so the first
test catches it.) The check uses only `[`, so it works in all three.

`#!/usr/bin/env bash` runs whichever `bash` comes first in `PATH`. On this
Mac that is only `/bin/bash`. If you install a newer bash elsewhere, a job
started with a short `PATH` (launchd, cron, another script) can still get
`/bin/bash`; the check turns that into a clear exit 1. I only ran it on
3.2 here; I didn't have bash 4 or 5 installed to run the passing side.

## Fix 2: write it for bash 3.2

For a short script it's often simpler to not need bash 4. This one runs on
macOS `/bin/bash` and covers the lookup, reading lines into an array, case
conversion and the last element:

```console
$ printf 'one\ntwo\nthree' > list.txt
$ cat > compat.sh <<'EOF'
#!/bin/bash
# Runs on macOS /bin/bash 3.2: no declare -A, mapfile, ${x,,} or ${a[-1]}.

color_of() {
  case $1 in
    apple) echo red ;;
    sky)   echo blue ;;
    *)     return 1 ;;
  esac
}

lines=()
while IFS= read -r line || [ -n "$line" ]; do
  lines+=("$line")
done < "$1"

echo "apple is $(color_of apple)"
echo "grass is $(color_of grass || echo unknown)"
echo "${#lines[@]} lines, last: ${lines[@]: -1}"
name=Hello
echo "lower: $(printf '%s' "$name" | tr '[:upper:]' '[:lower:]')"
echo "upper: $(printf '%s' "$name" | tr '[:lower:]' '[:upper:]')"
EOF
$ bash compat.sh list.txt; echo "exit $?"
apple is red
grass is unknown
3 lines, last: three
lower: hello
upper: HELLO
exit 0
```

Details that matter:

- **`|| [ -n "$line" ]`** keeps a last line that has no newline at the end
  (`list.txt` above has none). Without it, `read` fails on that line and the
  loop drops it:

  ```console
  $ bash -c 'n=0; while IFS= read -r l; do n=$((n+1)); done < list.txt; echo "$n lines"'
  2 lines
  ```

- **`${a[@]: -1}`** (with the space) is the last element in 3.2. Without
  the space, `:-1` is the "default value" expansion and you get the whole
  array. `${a[${#a[@]}-1]}` looks like it works too, but it's wrong for an
  array with gaps:

  ```console
  $ bash -c 'a=(x y z); echo "${a[@]: -1}"; echo "${a[@]:-1}"'
  z
  x y z
  $ bash -c 'a=([0]=x [5]=z); echo "[${a[${#a[@]}-1]}]"; echo "${a[@]: -1}"'
  []
  z
  ```

- **`tr '[:upper:]' '[:lower:]'`** follows the locale. With a UTF-8 locale
  `Ä` becomes `ä`; under `LC_ALL=C` it is left alone:

  ```console
  $ printf 'ÄÖ Straße\n' | LC_ALL=en_US.UTF-8 tr '[:upper:]' '[:lower:]'
  äö straße
  $ printf 'ÄÖ Straße\n' | LC_ALL=C tr '[:upper:]' '[:lower:]'
  ÄÖ straße
  ```

- **`**` (globstar)**: use `find . -name '*.txt'` instead.
- **`|&`**: write `cmd 2>&1 | other`.
- A `case` function only works for keys you know in advance. For keys that
  come from data, a different tool fits better: zsh, which ships with
  macOS, has associative arrays built in:

  ```console
  $ zsh -fc 'typeset -A color; color[apple]=red; color[sky]=blue; echo "apple is ${color[apple]}"'
  apple is red
  ```

## Clean up

```console
$ cd ~ && rm -rf "${T:?}"; ls -d "$T"
ls: /var/folders/xx/xxxxxxxx/T/tmp.XXXXXXXXXX: No such file or directory
```

(The folder name is random; I replaced the random parts with `x`/`X`.)

## Checklist

- `declare: -A: invalid option` on a Mac means the script ran under bash
  3.2 (`bash --version`). The script **keeps running**: the array becomes an
  indexed array, word keys are arithmetic (usually 0), the last assignment
  wins, and the exit status can still be 0.
- `mapfile`/`readarray` "command not found", `${x,,}`/`${x^^}` "bad
  substitution", `${a[-1]}` "bad array subscript", `globstar` "invalid shell
  option name", `|&` syntax error, `declare -n` invalid option: same cause.
- `bash -n` passed `declare -A`; only `|&` was a syntax error (the others were not run through it).
- `set -e` turns the failing `declare` or `mapfile` into a stop (exit 2 /
  127) instead of a wrong answer.
- If you need bash 4, check `BASH_VERSINFO` on the first lines and
  `exit 1`.
- Or write for 3.2: `case` lookup, `while IFS= read -r line || [ -n "$line" ]`,
  `tr`, `${a[@]: -1}`, `find`, `2>&1 |` — or use zsh's `typeset -A`.

*Tested on 2026-11-05 on macOS 26.5.2 (build 25F84, Apple silicon) with
`/bin/bash` 3.2.57 (also as `/bin/sh`), zsh 5.9, and `/usr/bin/tr`;
`PATH=/usr/bin:/bin:/usr/sbin:/sbin`. Not tested: bash 4/5 (Homebrew or
otherwise; the passing side of the version check was not run), Linux,
other locales than C and en_US.UTF-8.*
