---
title: "macOS sed: \\s, \\d, \\+ not working — `s/\\s+$//` turns \"boss\" into \"bo\" (exit 0)"
description: >-
  On macOS, /usr/bin/sed reads \s, \d, \w, \b as the plain letters s, d, w, b,
  and \+, \?, \| as plain +, ?, |. So the usual "trim trailing whitespace"
  line `sed -E 's/\s+$//'` leaves the spaces and deletes trailing s's instead
  ("boss" becomes "bo"), `s/[0-9]\+/N/` changes nothing, and `[ \t]` also
  matches the letter t. Every one of them exits 0, and `sed -i` writes the
  damage to disk. grep on the same Mac accepts these escapes. Use POSIX
  classes ([[:space:]], [0-9], [[:<:]]) with -E, or give sed -H, which turns
  the escapes on.
tags: [macos, sed, regex, shell]
---

A line copied from a "strip trailing whitespace" answer, run on a Mac:

```console
$ T=$(mktemp -d) && cd "$T"
$ printf 'class   \nboss\nyes\t\nid=42 size=7\n' > notes.txt
$ cat -et notes.txt
class   $
boss$
yes^I$
id=42 size=7$
$ sed -E 's/\s+$//' notes.txt | cat -et
class   $
bo$
yes^I$
id=42 size=7$
```

(`cat -et` shows the end of each line as `$` and a tab as `^I`.)

The spaces after `class` and the tab after `yes` are still there. Instead,
`boss` lost its last two letters. There's no error and no warning. With
`-i` the result goes straight into the file:

```console
$ cp notes.txt orig.txt
$ sed -i '' -E 's/\s+$//' notes.txt; echo "exit $?"
exit 0
$ cat -et notes.txt
class   $
bo$
yes^I$
id=42 size=7$
$ cp orig.txt notes.txt
```

The examples write small files (`notes.txt`, `orig.txt`, …) into the current
folder; the first line moves into a fresh temporary one, and the last
section removes it.

## What macOS sed does with `\s`

The macOS sed reads `\s` as a plain `s`. So `\s+$` means "one or more `s`
at the end of the line", which is exactly what it removed from `boss`. The
same happens to the other shortcuts you see in Linux answers:

```console
$ sed 's/[0-9]\+/N/g' notes.txt | tail -n 1
id=42 size=7
$ sed -E 's/\d+/N/g' notes.txt | tail -n 1
iN=42 size=7
$ echo 'cat catalog' | sed -E 's/\bcat\b/dog/g'
cat catalog
$ echo 'cat catalog' | sed -E 's/\<cat\>/dog/g'
cat catalog
$ echo 'a.jpg b.png' | sed 's/jpg\|png/img/g'; echo "exit $?"
a.jpg b.png
exit 0
```

- `[0-9]\+` is "a digit followed by a `+` sign": nothing matched.
- `\d+` is "one or more `d`": the `d` in `id` became `N`, the numbers stayed.
- `\bcat\b` is "`bcatb`": no match. `\<cat\>` didn't match either.
- `jpg\|png` is the literal text `jpg|png`: no match.

What each escape turned into, one line at a time:

```console
$ printf 'sun dw a+b a|b\n' | sed -E 's/\s/_/g'
_un dw a+b a|b
$ printf 'sun dw a+b a|b\n' | sed -E 's/\d/_/g'
sun _w a+b a|b
$ printf 'sun dw a+b a|b\n' | sed -E 's/\w/_/g'
sun d_ a+b a|b
$ printf 'sun dw a+b a|b\n' | sed 's/a\+/_/g'
sun dw _b a|b
$ printf 'sun dw a+b a|b\n' | sed 's/a\|b/_/g'
sun dw a+b _
```

There's one more trap, in the common BRE form of the same trim:

```console
$ printf 'salt\t\nbat \n' | sed 's/[ \t]*$//' | cat -et
salt^I$
ba$
```

Inside brackets, `\t` is not a tab here: `[ \t]` is the set "space,
backslash, `t`". So the tab after `salt` stayed and the `t` of `bat` went
with the space. (Outside brackets, `\t` and `\n` did work on this Mac:
`printf 'a\tb\n' | sed 's/\t/,/'` printed `a,b`.)

grep on the same Mac accepts these escapes, which makes the difference easy
to miss when you test a pattern with grep first:

```console
$ grep -cE '\s+$' notes.txt
2
$ echo 'cat catalog' | grep -oE '\bcat\b'
cat
```

The reason is in `man re_format`. `\d`, `\s`, `\w`, `\b`, `\<`, `\>` and
BRE `\+ \? \|` are listed under **ENHANCED FEATURES**, which are only on
when the program asks for them:

```text
ENHANCED FEATURES
     When the REG_ENHANCED flag is passed to one of the regcomp() variants,
     additional features are activated.
```

Going by the output above, grep gets them and sed, by default, doesn't.
`man sed` has the switch:

```text
     -H      Enable enhanced features in the regular expression syntax.  Note
             that this option is independent of the -E option.  See
             re_format(7) for details.
```

## Fix 1: POSIX classes (no escapes)

These work on this sed with no extra option:

- `\s` → `[[:space:]]` (or `[[:blank:]]` for space and tab only)
- `\d` → `[0-9]` or `[[:digit:]]`
- `\w` → `[[:alnum:]_]`
- `\bcat\b`, `\<cat\>` → `[[:<:]]cat[[:>:]]`
- BRE `\+`, `\?`, `\|` → `sed -E` with plain `+`, `?`, `|`
- `[ \t]` → `[[:blank:]]`

```console
$ sed -E 's/[[:space:]]+$//' notes.txt | cat -et
class$
boss$
yes$
id=42 size=7$
$ sed -E 's/[0-9]+/N/g' notes.txt | tail -n 1
id=N size=N
$ echo 'cat catalog' | sed -E 's/[[:<:]]cat[[:>:]]/dog/g'
dog catalog
$ echo 'a.jpg b.png' | sed -E 's/jpg|png/img/g'
a.img b.img
$ printf 'salt\t\nbat \n' | sed 's/[[:blank:]]*$//' | cat -et
salt$
bat$
```

`[[:space:]]`, `[0-9]` and `-E` with `+ ? |` are POSIX. `[[:<:]]` and
`[[:>:]]` are documented in this Mac's `man re_format`; I did not check
whether other seds accept them.

## Fix 2: `sed -H` (macOS only)

If you'd rather keep the pattern as written, add `-H`. It works with or
without `-E`, and with `-i ''`:

```console
$ sed -EH 's/\s+$//' notes.txt | cat -et
class$
boss$
yes$
id=42 size=7$
$ sed -H 's/[0-9]\+/N/g' notes.txt | tail -n 1
id=N size=N
$ echo 'cat catalog' | sed -EH 's/\bcat\b/dog/g'
dog catalog
$ cp orig.txt n2.txt; sed -i '' -EH 's/\s+$//' n2.txt; cat -et n2.txt
class$
boss$
yes$
id=42 size=7$
```

`-H` comes from the macOS `man sed`. I did not test it on other systems, so
don't use it in a script that also has to run elsewhere — use Fix 1 there.

## Fix 3: perl, with `-l`

`perl` ships with macOS and reads `\s`, `\d`, `\b` the way the Linux answers
mean. Use `-l`, though. Without it, `\s+$` also matches the newline, and the
lines run together:

```console
$ perl -pe 's/\s+$//' notes.txt | cat -et; echo
classbossyesid=42 size=7
$ perl -lpe 's/\s+$//' notes.txt | cat -et
class$
boss$
yes$
id=42 size=7$
```

(The `echo` only adds the line break that the first output no longer has.)

## Find the lines that need changing

A rough check for scripts you already have. It prints every line where
`sed` is followed by one of these escapes:

```console
$ cat > gnu-sed-escapes.sh <<'EOF'
#!/bin/sh
# Print sed lines that use \s \d \w \b \< \> \+ \? \| (and \S \D \W \B).
# macOS sed reads these as plain characters unless it gets -H.
grep -nE 'sed .*\\[sSdDwWbB<>+?|]' "$@"
EOF
$ cat > tidy.sh <<'EOF'
#!/bin/sh
sed -E 's/\s+$//' "$1" > "$1.tmp"
sed -i '' 's/\t/,/g' "$1.tmp"
sed -E 's/[[:space:]]+$//' "$1"
sed 's/[0-9]\+/N/g' "$1"
EOF
$ sh gnu-sed-escapes.sh tidy.sh; echo "exit $?"
2:sed -E 's/\s+$//' "$1" > "$1.tmp"
5:sed 's/[0-9]\+/N/g' "$1"
exit 0
```

It is only a text search: it also flags lines that already use `-H`, it
misses a sed command split over several lines, and it does not catch
`[ \t]` — look for that one by hand. Exit 1 means it found nothing.

## Clean up

```console
$ cd ~ && rm -rf "${T:?}"; ls -d "$T"
ls: /var/folders/xx/xxxxxxxx/T/tmp.XXXXXXXXXX: No such file or directory
```

(The folder name is random; I replaced the random parts with `x`/`X`.)

## Checklist

- On a Mac, `sed` without `-H` reads `\s \d \w \b` (and `\S \D \W \B`) as
  plain letters and `\+ \? \|` as plain characters. `\<` `\>` don't match.
  Nothing is printed and the exit status is 0.
- The visible symptom is usually "nothing changed" — or letters disappearing:
  `s/\s+$//` deletes trailing `s`, `\d+` replaces `d`.
- `[ \t]` is space, backslash and `t`, not space and tab.
- A pattern that works in `grep -E` on the same Mac can still fail in `sed`.
- Fix with POSIX classes (`[[:space:]]`, `[0-9]`, `[[:alnum:]_]`,
  `[[:blank:]]`, `[[:<:]]`/`[[:>:]]`) and `-E`; or `sed -H` on macOS only; or
  `perl -lpe` (keep the `-l`).
- Before `sed -i` on real files, run the same command without `-i` and look
  at the output with `cat -et`.

Related: [macOS `sed -i`: "invalid command code"](/2026/10/macos-sed-i-command-c-expects-invalid-command-code/)
and [macOS `grep -P` and lookarounds](/2026/10/macos-grep-invalid-option-p-lookahead-perl-ne/).

*Tested on 2026-11-12 on macOS 26.5.2 (build 25F84, Apple silicon) with
`/usr/bin/sed` (text_cmds-199), `/usr/bin/grep`, perl 5.34.1 and zsh 5.9;
`PATH=/usr/bin:/bin:/usr/sbin:/sbin`. Not tested: GNU sed, Linux, older
macOS versions, locales other than the default one here.*
