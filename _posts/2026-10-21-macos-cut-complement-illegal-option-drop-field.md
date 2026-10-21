---
title: "macOS `cut: illegal option -- -` for `--complement`: drop a field with `cut -f1-2,4-` or awk (and the `-f1-0` trap)"
description: >-
  `cut --complement -f3` fails on macOS with "cut: illegal option -- -"
  (exit 1) because /usr/bin/cut is BSD cut. Name the fields to keep with an
  open range (`-f1-2,4-`), which works without knowing the field count;
  dropping field 1 needs `-f2-`, because `-f1-0` is rejected with "values may
  not include zero". An awk loop does the same for any field, but `$3=""`
  leaves an empty column and the `NF--` shortcut ate fields on short lines.
  Also: cut splits on tab by default, `-d ' '` counts every space, `-w`
  joins output with tabs. Reproduced on macOS 26.
tags: [macos, shell, cut, awk, bsd]
---

A one-liner that removes a column from a CSV stops on a Mac:

```console
$ mkdir demo && cd demo
$ printf 'id,name,email,city\n1,ann,a@example.com,Oslo\n2,bob,b@example.com,Lima\n' > users.csv
$ cut --complement -d, -f3 users.csv; echo "exit $?"
cut: illegal option -- -
usage: cut -b list [-n] [file ...]
       cut -c list [file ...]
       cut -f list [-s] [-w | -d delim] [file ...]
exit 1
```

Moving `--complement` after `-d,` gives the same error:

```console
$ cut -d, --complement -f3 users.csv; echo "exit $?"
cut: illegal option -- -
usage: cut -b list [-n] [file ...]
       cut -c list [file ...]
       cut -f list [-s] [-w | -d delim] [file ...]
exit 1
```

`--complement` is a
GNU cut option (I didn't run GNU cut here). macOS ships BSD cut; its usage
text above lists no long options, and the man page doesn't mention a
complement:

```console
$ what /usr/bin/cut
/usr/bin/cut:
	PROGRAM:cut  PROJECT:text_cmds-199
	PROGRAM:cut  PROJECT:text_cmds-199
$ man cut | col -b | grep -ci complement
0
```

## Fix 1: name the fields you keep, with an open range

`cut -f` takes a list, and a range may be open on the right. To drop field 3,
keep 1–2 and 4-to-the-end:

```console
$ cut -d, -f1-2,4- users.csv
id,name,city
1,ann,Oslo
2,bob,Lima
```

`4-` means "field 4 and everything after", so this doesn't need the field
count. The same holds on lines with more or fewer fields:

```console
$ printf 'a,b,c,d,e,f\nx,y\nnodelim\n' > ragged.csv
$ cut -d, -f1-2,4- ragged.csv
a,b,d,e,f
x,y
nodelim
```

The last line has no comma, so cut printed it unchanged. That's documented:
"Unless specified, lines with no delimiters are passed through unmodified."
Add `-s` to drop such lines instead:

```console
$ cut -s -d, -f1-2,4- ragged.csv
a,b,d,e,f
x,y
```

The list doesn't reorder fields. `-f4,1` still prints them in input order:

```console
$ cut -d, -f4,1 users.csv
id,city
1,Oslo
2,Lima
```

### The `-f1-0` trap when dropping field 1

If you build the list as `1-(N-1),(N+1)-`, N=1 gives `1-0`, and cut rejects it:

```console
$ cut -d, -f1-0,2- users.csv; echo "exit $?"
cut: [-bcf] list: values may not include zero
exit 1
$ cut -d, -f-0,2- users.csv; echo "exit $?"
cut: [-bcf] list: values may not include zero
exit 1
$ cut -d, -f2- users.csv
name,email,city
ann,a@example.com,Oslo
bob,b@example.com,Lima
```

A small function that handles both cases:

```sh
cut_drop() {  # usage: cut_drop FIELD DELIM [file ...]
  n=$1 d=$2; shift 2
  if [ "$n" -eq 1 ]; then
    cut -d "$d" -f 2- "$@"
  else
    cut -d "$d" -f "1-$((n - 1)),$((n + 1))-" "$@"
  fi
}
```

```console
$ cut_drop 1 , users.csv
name,email,city
ann,a@example.com,Oslo
bob,b@example.com,Lima
$ cut_drop 3 , users.csv
id,name,city
1,ann,Oslo
2,bob,Lima
$ cut_drop 1 , ragged.csv
b,c,d,e,f
y
nodelim
```

The `nodelim` line is still passed through — even when you "drop" its only
field.

## Fix 2: awk, and two awk shortcuts that go wrong

Setting the field to empty does not remove it. The separator stays:

```console
$ awk -F, -v OFS=, '{ $3 = "" } 1' users.csv
id,name,,city
1,ann,,Oslo
2,bob,,Lima
```

Build the line from the fields you keep instead:

```console
$ awk -F, -v OFS=, -v c=3 '{ o = ""; s = ""; for (i = 1; i <= NF; i++) if (i != c) { o = o s $i; s = OFS }; print o }' users.csv
id,name,city
1,ann,Oslo
2,bob,Lima
$ awk -F, -v OFS=, -v c=3 '{ o = ""; s = ""; for (i = 1; i <= NF; i++) if (i != c) { o = o s $i; s = OFS }; print o }' ragged.csv
a,b,d,e,f
x,y
nodelim
```

The `s` variable (empty before the first kept field, `OFS` after) keeps a
leading empty field intact. A version that checks `o == ""` to decide
whether to add a separator loses it:

```console
$ printf ',b,c\n' | awk -F, -v OFS=, -v c=3 '{ o = ""; s = ""; for (i = 1; i <= NF; i++) if (i != c) { o = o s $i; s = OFS }; print o }'
,b
$ printf ',b,c\n' | awk -F, -v OFS=, -v c=3 '{ o = ""; for (i = 1; i <= NF; i++) if (i != c) o = (o == "" ? $i : o OFS $i); print o }'
b
```

Unlike cut, awk doesn't pass `nodelim` through when you drop field 1 — the
line has one field, you removed it, and an empty line is left:

```console
$ awk -F, -v OFS=, -v c=1 '{ o = ""; s = ""; for (i = 1; i <= NF; i++) if (i != c) { o = o s $i; s = OFS }; print o }' ragged.csv
b,c,d,e,f
y

```

A shorter version you'll see around shifts the fields left and decrements
`NF`. In this awk it works on full lines but not on short ones:

```console
$ awk -F, -v OFS=, -v c=3 '{ for (i = c; i < NF; i++) $i = $(i + 1); NF--; print }' ragged.csv
a,b,d,e,f
x

```

`x,y` has no field 3, but `NF--` still removed the last field (`y`), and
`nodelim` became an empty line.

## Delimiters: where cut and awk differ

**Default is tab, not whitespace.** Without `-d`, cut splits on tab. A CSV
has no tabs, so every line is "without delimiter" and comes back whole,
with exit 0:

```console
$ cut -f2 users.csv; echo "exit $?"
id,name,email,city
1,ann,a@example.com,Oslo
2,bob,b@example.com,Lima
exit 0
```

**`-d ' '` counts every space.** Two spaces in a row make an empty field.
awk's default field splitting doesn't:

```console
$ printf 'web  8080 up\ndb 5432 down\n' > ps.txt
$ cut -d ' ' -f2 ps.txt

5432
$ awk '{ print $2 }' ps.txt
8080
5432
```

**BSD cut has `-w`.** From the man page: "Use whitespace (spaces and tabs)
as the delimiter. Consecutive spaces and tabs count as one single field
separator." The output fields are joined with a tab:

```console
$ cut -w -f2 ps.txt
8080
5432
$ cut -w -f1,3- ps.txt | od -c
0000000    w   e   b  \t   u   p  \n   d   b  \t   d   o   w   n  \n
0000017
```

**awk's default rebuild uses single spaces.** Once you assign to a field
with the default `FS`, awk rejoins the line with `OFS` (a space), so the
original spacing is gone, and the emptied field leaves two spaces:

```console
$ awk '{ $2 = "" } 1' ps.txt | od -c
0000000    w   e   b           u   p  \n   d   b           d   o   w   n
0000020   \n
0000021
```

**One character only.** cut won't take a multi-character delimiter:

```console
$ cut -d ', ' -f1 users.csv; echo "exit $?"
cut: bad delimiter
exit 1
```

**Quoted CSV is not handled by either.** A comma inside quotes is still a
separator to both:

```console
$ printf '1,"Lee, Ann",x@example.org\n' | cut -d, -f1-2,4-
1,"Lee,x@example.org
$ printf '1,"Lee, Ann",x@example.org\n' | awk -F, '{ print NF }'
4
```

For real CSV with quoted fields, use a CSV parser (for example Python's
`csv` module) — not tested in this post.

## Checklist

- `cut: illegal option -- -` (exit 1 here): `/usr/bin/cut` on macOS is BSD
  cut; its usage text lists no long options and its man page has no
  complement.
- Drop field N: keep `1-(N-1),(N+1)-`. The open `-` end means you don't need
  the field count.
- Dropping field 1: use `-f2-`. `-f1-0` and `-f-0,2-` fail with "values may
  not include zero".
- Lines with no delimiter come through unchanged; `-s` drops them.
- `-f4,1` prints in input order, not 4 then 1.
- awk: `$3 = ""` leaves an empty column. Rebuild from kept fields; avoid
  `NF--` shortcuts on lines that may be short.
- cut default delimiter is tab; `-d ' '` counts every space; `-w` treats
  runs of whitespace as one and outputs tabs; a two-character `-d` is rejected.
- Neither cut nor `awk -F,` understands quoted CSV.

*Tested on macOS 26.5.2 (Apple silicon) with `/usr/bin/cut`
(`PROGRAM:cut  PROJECT:text_cmds-199`) and `/usr/bin/awk` (version 20200816)
in zsh 5.9 (arm64-apple-darwin25.0). GNU cut, gawk, bash, and Linux were not
tested; `cut -b`/`-c` were not tested.*
