---
title: "macOS `sh` prints `-e` / `-n` literally: `echo -e` in a `#!/bin/sh` script, and `C:\\new` turned into a newline — use `printf '%s\\n'`"
description: >-
  On macOS a `#!/bin/sh` script that runs `echo -e "name\tsize"` prints
  "-e name	size", and `echo -n "total: "` prints "-n total: " plus a newline.
  `/bin/sh` is bash 3.2 in POSIX mode, where `echo` takes no options and
  expanded the backslashes I tried; `bash script.sh` and `zsh script.sh` print it
  the way you expected. The same rule mangles data: a value containing
  `C:\new\temp` comes out split across lines from `echo "$p"` in `sh` and
  in zsh. `printf '%s\n' "$p"` printed it the same in sh, bash, zsh and dash.
  Reproduced on macOS 26.
tags: [macos, shell, sh, echo, printf, bash, zsh]
---

A small script with a `#!/bin/sh` line prints its own option flags:

```console
$ mkdir demo && cd demo
$ printf '%s\n' '#!/bin/sh' 'echo -e "name\tsize"' 'echo -n "total: "' 'echo 42' > report.sh
$ chmod +x report.sh
$ ./report.sh
-e name	size
-n total: 
42
```

The tab is there, so the `\t` was expanded — but `-e` was printed instead of
being read as an option, and `-n` didn't suppress the newline either. Run the
same file with bash or zsh and it looks fine, which is why it "works on my
machine" when someone tests it with `bash report.sh`:

```console
$ bash report.sh
name	size
total: 42
$ sh report.sh
-e name	size
-n total: 
42
$ zsh report.sh
name	size
total: 42
```

## Why: `/bin/sh` is bash in POSIX mode

On this Mac `/bin/sh` looks like a small launcher that runs the shell named by
`/private/var/select/sh`, which points at bash 3.2:

```console
$ ls -l /bin/sh /private/var/select/sh
-rwxr-xr-x  1 root  wheel  101232 Jun 25 11:29 /bin/sh
lrwxr-xr-x  1 root  wheel       9 Jun 25 11:29 /private/var/select/sh -> /bin/bash
$ /bin/sh --version | head -n 1
GNU bash, version 3.2.57(1)-release (arm64-apple-darwin25)
```

Started as `sh`, bash turns on POSIX mode, and in POSIX mode its `echo`
behaves the way the POSIX/XSI `echo` does: no `-e`, no `-n`, backslash
escapes always expanded. What I saw here: `shopt` reports `xpg_echo` as
*off*, and turning it off again doesn't change anything; only leaving POSIX
mode does. For `echo`, `bash --posix` behaved the same as `sh` here.

```console
$ /bin/sh -c 'set -o | grep posix; shopt xpg_echo'
posix          	on
xpg_echo       	off
$ /bin/sh -c 'shopt -u xpg_echo; echo -e "a\tb"'
-e a	b
$ /bin/sh -c 'set +o posix; echo -e "a\tb"'
a	b
$ bash --posix -c 'echo -e "a\tb"'
-e a	b
```

`set +o posix` is not a fix worth shipping — the script would then depend on
`/bin/sh` being bash. The `dash` that macOS also ships (Debian and Ubuntu, by general knowledge, not tested here,
use dash as `/bin/sh`) prints `-e` too:

```console
$ /bin/dash -c 'echo -e "a\tb"'
-e a	b
```

The shebang doesn't protect you if someone types `sh script`: `sh` reads the
file itself and the `#!/bin/bash` line is just a comment.

```console
$ printf '%s\n' '#!/bin/bash' 'echo -e "a\tb"' > b.sh
$ chmod +x b.sh
$ ./b.sh
a	b
$ sh b.sh
-e a	b
```

## Each shell expands backslashes differently

Without any option, the three shells already disagree about `\t`:

```console
$ /bin/sh -c 'echo "a\tb"'
a	b
$ bash -c 'echo "a\tb"'
a\tb
$ zsh -fc 'echo "a\tb"'
a	b
```

zsh's `echo` expands escapes by default (like `sh` here), bash's doesn't.
That matters more for data than for literals. A Windows path read from a
file goes through `echo "$p"` and comes out with a newline and a tab in it
— in `sh` *and* in zsh, the default login shell on macOS:

```console
$ printf '%s\n' 'C:\new\temp' > path.txt
$ /bin/sh -c 'p=$(cat path.txt); echo "$p"'
C:
ew	emp
$ bash -c 'p=$(cat path.txt); echo "$p"'
C:\new\temp
$ zsh -fc 'p=$(cat path.txt); echo "$p"'
C:
ew	emp
```

Quoting `"$p"` doesn't help: the shell passes the backslashes through
untouched, and it's `echo` that expands them.

## Fix: `printf`, with the data in `%s`

`printf` does the same thing in all of them. Put the format in the first
argument and the data in `%s` arguments:

```console
$ for s in /bin/sh bash zsh /bin/dash; do $s -c 'p=$(cat path.txt); printf "%s\n" "$p"'; done
C:\new\temp
C:\new\temp
C:\new\temp
C:\new\temp
```

Don't put the data into the format string. A `%` in the value is read as a
conversion — here `% d` (space flag, `d`) ate the space and the `d`, printed
` 0`, and the exit status was still 0:

```console
$ p='50% done'; printf "$p\n"; echo "exit $?"
50 0one
exit 0
$ printf '%s\n' "$p"
50% done
```

The script from the top, with `printf` replacing both `echo -e` and
`echo -n`, gives the same bytes in all four shells:

```console
$ printf '%s\n' '#!/bin/sh' 'printf "name\tsize\n"' 'printf "%s" "total: "' 'echo 42' > report2.sh
$ chmod +x report2.sh
$ for s in ./report2.sh "bash report2.sh" "zsh report2.sh" "/bin/dash report2.sh"; do eval "$s" | od -c | head -n 2; done
0000000    n   a   m   e  \t   s   i   z   e  \n   t   o   t   a   l   :
0000020        4   2  \n
0000000    n   a   m   e  \t   s   i   z   e  \n   t   o   t   a   l   :
0000020        4   2  \n
0000000    n   a   m   e  \t   s   i   z   e  \n   t   o   t   a   l   :
0000020        4   2  \n
0000000    n   a   m   e  \t   s   i   z   e  \n   t   o   t   a   l   :
0000020        4   2  \n
```

(`od` pads its last line with spaces; trimmed here.) `echo 42` is still
fine: plain text without a leading `-` or a backslash prints the same in the four shells I tried.
If you're writing zsh only, `print -r -- "$p"` or `echo -E "$p"`
printed the path unchanged too:

```console
$ zsh -fc 'p=$(cat path.txt); print -r -- "$p"; echo -E "$p"'
C:\new\temp
C:\new\temp
```

## Checklist

- `-e` or `-n` showing up in a script's output on macOS: the script runs
  under `/bin/sh` (shebang `#!/bin/sh`, or started as `sh script`), which is
  bash 3.2 in POSIX mode. Its `echo` takes no options and expanded the `\t` and `\n` I tried
  backslashes.
- `shopt -u xpg_echo` didn't change that here; `set +o posix` did, but don't
  rely on `/bin/sh` being bash — `/bin/dash` prints `-e` as well.
- `sh script` ignores a `#!/bin/bash` line.
- Values with backslashes (like `C:\new`) are mangled by
  `echo "$var"` in `sh` and in zsh, not in bash.
- Use `printf '%s\n' "$var"` for data, `printf '%s' "..."` instead of
  `echo -n`, and `printf 'a\tb\n'` instead of `echo -e`.
- Never `printf "$var"`: a `%` in the value is a conversion (`50% done` →
  `50 0one`, exit 0).
- zsh-only scripts: `print -r -- "$var"` or `echo -E "$var"`.

*Tested on macOS 26.5.2 (Apple M1): `/bin/sh` → GNU bash 3.2.57(1)-release
in POSIX mode, `/bin/bash` 3.2.57, zsh 5.9 (arm64-apple-darwin25.0),
`/bin/dash` (dash-16). All console lines replayed from this post in one
`zsh -f` session in a temporary directory. Linux, newer bash versions, and
GNU coreutils' `/bin/echo` and `/usr/bin/printf` were not tested — only the
shells' built-in `echo` and `printf`.*
