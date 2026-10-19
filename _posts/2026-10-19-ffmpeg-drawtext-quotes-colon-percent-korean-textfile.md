---
title: "ffmpeg drawtext breaks on quotes, colons, commas and %: \"No option name near\", \"Stray %\", blank or □□□ text — use textfile= with expansion=none"
description: >-
  Burning a caption with ffmpeg's drawtext failed as soon as the text had an
  apostrophe, a colon or a comma ("No option name near '30:fontsize=48...'"),
  and a % made ffmpeg exit 0 with no text at all ("Stray % near ''"). Korean
  came out as boxes until fontfile= pointed at a Korean font. The text goes
  through three parsers; textfile= skips two of them, and expansion=none
  skips the third. Also: zsh eats "$FONT:text" as a :t modifier.
tags: [ffmpeg, macos, video, python]
---

You add a caption to a video with ffmpeg's `drawtext` filter, and it works —
until the caption is a real sentence. An apostrophe, a time like `3:30`, or a
comma gives **`No option name near '30:fontsize=48:...'`** and exit code 234.
A **`%` is worse: ffmpeg prints `Stray % near ''`, exits 0, and the video has
no caption at all.** And Korean (or any non-Latin text) may come out as a row
of empty boxes.

Below is each failure reproduced on my Mac, a working way to escape the text
if you must use `text=`, and the fix I ended up with: put the caption in a file
and use `textfile=` **with `expansion=none`** — `textfile=` alone still fails on `%`.

Every command here renders one frame from a generated black background
(`-f lavfi -i color=...`) to a PNG, so you can try them without a video. `-y`
overwrites the output file without asking — run them in an empty folder.

## The failures

**No font that has Korean glyphs: boxes.** Without `fontfile=`, the static
ffmpeg build I used couldn't load a fontconfig configuration, fell back to a
font without Hangul, and drew one box per syllable. Exit code 0:

```console
$ ffmpeg -hide_banner -loglevel error -f lavfi -i color=c=black:s=800x160:d=1 -vf "drawtext=text='안녕하세요':fontsize=48:fontcolor=white:x=20:y=50" -frames:v 1 -y 1-tofu.png; echo "exit $?"
Fontconfig error: Cannot load default config file: No such file: (null)
Fontconfig error: Cannot load default config file: No such file: (null)
exit 0
```

The PNG shows `□□□□□`. Pointing `fontfile=` at a font with Korean glyphs
fixes it; macOS ships one at `/System/Library/Fonts/AppleSDGothicNeo.ttc`. (A
Homebrew ffmpeg with a working fontconfig setup may pick a different default
font; I only tested this build.)

**An apostrophe inside `text='...'`:**

```console
$ ffmpeg -hide_banner -loglevel error -f lavfi -i color=c=black:s=800x160:d=1 -vf "drawtext=fontfile=/System/Library/Fonts/AppleSDGothicNeo.ttc:text='it's 오후 3:30':fontsize=48:fontcolor=white:x=20:y=50" -frames:v 1 -y 2-quote.png; echo "exit $?"
[AVFilterGraph @ 0x6000039ec000] No option name near '30:fontsize=48:fontcolor=white:x=20:y=50'
[AVFilterGraph @ 0x6000039ec000] Error parsing a filter description around: 
[AVFilterGraph @ 0x6000039ec000] Error parsing filterchain 'drawtext=fontfile=/System/Library/Fonts/AppleSDGothicNeo.ttc:text='it's 오후 3:30':fontsize=48:fontcolor=white:x=20:y=50' around: 
Error opening output file 2-quote.png.
Error opening output files: Invalid argument
exit 234
```

The `'` in `it's` closes the quote, so the `:` in `3:30` is no longer quoted and
ends the `text` option. ffmpeg then reads `30` as the next option, and it has
no `name=`. Without quotes, a plain `text=오후 3:30` fails with the same message.

**A comma:**

```console
$ ffmpeg -hide_banner -loglevel error -f lavfi -i color=c=black:s=800x160:d=1 -vf "drawtext=fontfile=/System/Library/Fonts/AppleSDGothicNeo.ttc:text=네, 맞아요:fontsize=48:fontcolor=white:x=20:y=50" -frames:v 1 -y 3-comma.png; echo "exit $?"
[AVFilterGraph @ 0x600001d8a4c0] No option name near '48:fontcolor=white:x=20:y=50'
[AVFilterGraph @ 0x600001d8a4c0] Error parsing a filter description around: 
[AVFilterGraph @ 0x600001d8a4c0] Error parsing filterchain 'drawtext=fontfile=/System/Library/Fonts/AppleSDGothicNeo.ttc:text=네, 맞아요:fontsize=48:fontcolor=white:x=20:y=50' around: 
Error opening output file 3-comma.png.
Error opening output files: Invalid argument
exit 234
```

In a filtergraph a comma separates filters, so everything after `네` became a
second "filter".

**A percent sign — the silent one:**

```console
$ ffmpeg -hide_banner -loglevel error -f lavfi -i color=c=black:s=800x160:d=1 -vf "drawtext=fontfile=/System/Library/Fonts/AppleSDGothicNeo.ttc:text='할인 50%':fontsize=48:fontcolor=white:x=20:y=50" -frames:v 1 -y 4-percent.png; echo "exit $?"
[Parsed_drawtext_0 @ 0x6000006802c0] Stray % near ''
exit 0
```

Exit 0, the file is written, and the frame is **plain black: the whole caption
is dropped, not just the `%`.** drawtext treats `%{...}` as an expansion (for
things like `%{pts}`), and a `%` that doesn't start one makes it give up on the
text. If your script only checks the exit code, it won't notice.

## Why: three parsers

The caption is parsed three times before it's drawn, and each parser has its
own special characters:

1. **The filtergraph** (`-vf`): `,` `;` `[` `]` separate filters and links;
   `'` quotes; `\` escapes.
2. **The filter's options**: `:` separates options; `'` and `\` again.
3. **drawtext's text expansion**: `%` starts `%{...}`; `\` escapes.

(The shell is a fourth if you type the command, which is why the escaped
`text=` versions you find online are hard to read.)

## If you must use `text=`: escape for each parser

Escape in the reverse order the parsers read the text: drawtext first, then
the option value, then the filtergraph. This Python version passes the filter
as a list item to `subprocess.run`, so there's no shell layer:

```python
# escaped.py -- the text= route: escape for three parsers, in this order
import subprocess

FONT = "/System/Library/Fonts/AppleSDGothicNeo.ttc"

def drawtext_escape(s):
    s = s.replace("\\", "\\\\").replace("%", "\\%")              # 1. drawtext's % expansion
    s = s.replace("\\", "\\\\").replace(":", "\\:").replace("'", "\\'")  # 2. option value
    for c in "\\'[],;":                                          # 3. filtergraph
        s = s.replace(c, "\\" + c)
    return s

caption = "it's 오후 3:30, 할인 50% [끝]"
vf = (f"drawtext=fontfile={FONT}:text={drawtext_escape(caption)}"
      ":fontsize=48:fontcolor=white:x=20:y=50")
print(vf)
subprocess.run(["ffmpeg", "-hide_banner", "-loglevel", "error",
                "-f", "lavfi", "-i", "color=c=black:s=800x160:d=1",
                "-vf", vf, "-frames:v", "1", "-y", "escaped.png"], check=True)
```

```console
$ python3 escaped.py
drawtext=fontfile=/System/Library/Fonts/AppleSDGothicNeo.ttc:text=it\\\'s 오후 3\\:30\, 할인 50\\\\% \[끝\]:fontsize=48:fontcolor=white:x=20:y=50
```

That rendered `it's 오후 3:30, 할인 50% [끝]` correctly, and so did
`"따옴표" C:\temp; a=b` (quotes, a backslash, a semicolon). It works, but look
at `50\\\\%`: four backslashes for one percent sign. It's easy to get one layer
wrong, and the `%` mistake doesn't raise an error.

## The fix: `textfile=` plus `expansion=none`

Write the caption to a file and give drawtext the file's path. The filtergraph
and option parsers only see the path; the caption itself is never parsed by them.

But `textfile=` alone **still fails on `%`**, because drawtext's expansion also
runs on text read from the file:

```console
$ printf '%s' "it's 오후 3:30, 할인 50% [끝]" > caption.txt
$ ffmpeg -hide_banner -loglevel error -f lavfi -i color=c=black:s=800x160:d=1 -vf "drawtext=fontfile=/System/Library/Fonts/AppleSDGothicNeo.ttc:textfile=caption.txt:fontsize=48:fontcolor=white:x=20:y=50" -frames:v 1 -y 7-textfile.png; echo "exit $?"
[Parsed_drawtext_0 @ 0x600002720fd0] Stray % near ' [끝]'
exit 0
```

Black frame again. Add `expansion=none` and the text is drawn exactly as it is
in the file:

```console
$ ffmpeg -hide_banner -loglevel error -f lavfi -i color=c=black:s=800x160:d=1 -vf "drawtext=fontfile=/System/Library/Fonts/AppleSDGothicNeo.ttc:textfile=caption.txt:expansion=none:fontsize=48:fontcolor=white:x=20:y=50" -frames:v 1 -y 8-textfile-none.png; echo "exit $?"
exit 0
```

The frame shows `it's 오후 3:30, 할인 50% [끝]`. (`expansion=none` also turns
off `%{pts}` and friends; if you need a timestamp in the same caption, that
needs `text=` and escaping.)

In a script, a temporary file per caption:

```python
# textfile.py -- the textfile= route: no escaping of the caption at all
import os, subprocess, tempfile

FONT = "/System/Library/Fonts/AppleSDGothicNeo.ttc"

def render(caption, out):
    fd, path = tempfile.mkstemp(suffix=".txt")   # e.g. /var/folders/.../T/tmpab12.txt
    try:
        with os.fdopen(fd, "w", encoding="utf-8") as f:
            f.write(caption)
        vf = (f"drawtext=fontfile={FONT}:textfile={path}:expansion=none"
              ":fontsize=48:fontcolor=white:x=20:y=50")
        subprocess.run(["ffmpeg", "-hide_banner", "-loglevel", "error",
                        "-f", "lavfi", "-i", "color=c=black:s=800x160:d=1",
                        "-vf", vf, "-frames:v", "1", "-y", out], check=True)
    finally:
        os.remove(path)

render("it's 오후 3:30, 할인 50% [끝]", "textfile.png")
render('"따옴표" C:\\temp; a=b', "textfile2.png")
print("ok")
```

```console
$ python3 textfile.py
ok
```

Both frames showed the captions exactly, with no escaping code. Write the file
as UTF-8 (`encoding="utf-8"`). A trailing newline (`echo` instead of
`printf '%s'`) made no visible difference with this ffmpeg.

**The path is still parsed.** The file name goes through the same filtergraph
and option parsers, so a `:` in it breaks the command the same way:

```console
$ printf '%s' "안녕" > "a:b.txt"
$ ffmpeg -hide_banner -loglevel error -f lavfi -i color=c=black:s=640x160:d=1 -vf "drawtext=fontfile=/System/Library/Fonts/AppleSDGothicNeo.ttc:textfile=a:b.txt:expansion=none:fontsize=48:fontcolor=white:x=20:y=50" -frames:v 1 -y colon.png; echo "exit $?"
[AVFilterGraph @ 0x6000005a0230] No option name near 'b.txt:expansion=none:fontsize=48:fontcolor=white:x=20:y=50'
...
exit 234
```

Use paths you control, like the ones `tempfile.mkstemp()` returns, rather than
a name built from the caption.

## zsh trap: `$FONT:text` is not what you typed

On macOS the default shell is zsh. Putting the font path in a variable and
writing `fontfile=$FONT:text=...` fails with a different error:

```console
$ FONT=/System/Library/Fonts/AppleSDGothicNeo.ttc
$ ffmpeg -hide_banner -loglevel error -f lavfi -i color=c=black:s=800x160:d=1 -vf "drawtext=fontfile=$FONT:text='안녕하세요':fontsize=48:fontcolor=white:x=20:y=50" -frames:v 1 -y 5-zsh.png; echo "exit $?"
[Parsed_drawtext_0 @ 0x600003a84420] Either text, a valid file, a timecode or text source must be provided
[AVFilterGraph @ 0x600002684380] Error initializing filters
Error opening output file 5-zsh.png.
Error opening output files: Invalid argument
exit 234
```

zsh read `$FONT:t` as "the last path component of `$FONT`" (the `:t` modifier),
so ffmpeg received `fontfile=AppleSDGothicNeo.ttcext='안녕하세요'` — no font
path and no `text` option. Write `${FONT}:text=...` and it works (`exit 0`, text
drawn). bash doesn't have this problem, but braces work in both.

## Checklist

- Korean shows as boxes: give drawtext `fontfile=` a font that has the glyphs
  (`/System/Library/Fonts/AppleSDGothicNeo.ttc` on macOS).
- `No option name near '...'` with a caption: a `:`, `'` or `,` in the text
  ended the option or the filter early.
- `Stray % near ...` with exit 0: the whole caption was dropped. Don't rely on
  the exit code; look at a frame.
- Fix: write the caption to a file and use `textfile=<path>:expansion=none`.
  `textfile=` without `expansion=none` still fails on `%`.
- Keep the text file's path free of `:`, `'`, `,` (a temp file is fine).
- If you need `text=`, escape for drawtext, then the option, then the
  filtergraph, and pass the filter as one argv item (no shell).
- In zsh, write `${VAR}:` before a colon, not `$VAR:`.

*Tested on macOS 26.5.2 (Apple silicon) with ffmpeg 7.1, the static arm64
build shipped in the imageio-ffmpeg 0.6.0 Python package (built with
libfreetype, libharfbuzz and fontconfig), from zsh; Python scripts run with
Python 3.8.2 (`/usr/bin/python3`). The font-fallback result is specific to
that build; filter addresses like `0x6000039ec000` differ between runs.*
