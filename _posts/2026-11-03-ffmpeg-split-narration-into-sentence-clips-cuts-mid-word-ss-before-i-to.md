---
title: "ffmpeg: splitting a narration into sentence clips cuts words off, and -ss before -i makes -to clips too long"
description: >-
  Cutting a narration into one clip per sentence at times estimated from
  the script's character counts put the cuts inside words (on a generated
  test tone file): a clip started
  27 ms into its first word and ended with 25 ms of the next one. Snap
  each estimate to the longest silencedetect pause near it and cut in the
  middle of the pause. Also: with -ss before -i, an output -to counts from
  the new start, so the clip came out 4.70 s instead of 2.44 s (exit 0).
  ffmpeg 7.1.
tags: [ffmpeg, audio, tts, macos]
---

You have one narration file from a text-to-speech engine and the script it
was made from, and you want one clip per sentence. The quick way is to guess
each sentence's end from its share of the characters and cut there. ffmpeg
exits 0 and the clips have about the right length, but **a clip can start a few
milliseconds into its first word, and the clip before it ends with the start
of that word.** And if you put `-ss` before `-i` to speed up the cut, **`-to`
stops meaning "until this time" and the clip comes out too long** (here it
ran to the end of the file), again with exit 0.

Below I reproduce both with a generated narration whose sentence boundaries
I know, then give a script that moves each guess into a real pause. (The
previous post is about picking the `silencedetect` threshold:
[ffmpeg silencedetect prints nothing, or cuts off word endings]({% post_url 2026-11-02-ffmpeg-silencedetect-no-output-cuts-word-endings-shorten-tts-pauses %}).)

```console
$ ffmpeg -version | head -1
ffmpeg version 7.1 Copyright (c) 2000-2024 the FFmpeg developers
```

Everything below runs in one empty folder, in this order.

## A test narration with known sentence boundaries

Real TTS would do, but then I can't say exactly where a sentence ends. This
script writes a 24 kHz mono WAV in which every word is a tone (longer words
are longer tones), with 0.06 s between words, 0.45 s after a comma and 0.6 s
between sentences, over a noise floor:

```python
#!/usr/bin/env python3
"""make-narration.py: a TTS-like narration with known sentence boundaries.

Writes narration.wav (24 kHz mono, 16-bit) and script.txt (one sentence per
line), and prints where each sentence really starts and ends.
Words are tones; gaps between words 0.06 s, after a comma 0.45 s, between
sentences 0.6 s. Noise floor peaks at -54 dB.
"""
import math, random, struct, wave

SENTENCES = [
    "The cache was empty.",
    "So every request went to the disk, and the disk was slow.",
    "We added a warm-up step.",
    "Now it starts fast.",
]
RATE = 24000
random.seed(1)
samples, t = [], 0.0

def silence(sec):
    global t
    samples.extend([0.0] * int(sec * RATE)); t += sec

def word(w, freq):
    global t
    n = int((0.1 + 0.055 * len(w)) * RATE)
    fade = int(0.01 * RATE)
    for i in range(n):
        env = min(1.0, i / fade, (n - i) / fade)
        samples.append(0.4 * env * math.sin(2 * math.pi * freq * i / RATE))
    t += n / RATE

silence(0.3)
with open("script.txt", "w") as f:
    for k, s in enumerate(SENTENCES):
        f.write(s + "\n")
        start = t
        words = s.split()
        for j, w in enumerate(words):
            word(w, 200 + 15 * j)
            if j < len(words) - 1:
                silence(0.45 if w.endswith(",") else 0.06)
        print(f"sentence {k + 1}: {start:.3f} .. {t:.3f}")
        silence(0.6 if k < len(SENTENCES) - 1 else 0.5)

with wave.open("narration.wav", "wb") as w:
    w.setnchannels(1); w.setsampwidth(2); w.setframerate(RATE)
    w.writeframes(b"".join(
        struct.pack("<h", int(32767 * (x + 0.002 * (2 * random.random() - 1))))
        for x in samples))
print(f"total {len(samples) / RATE:.3f} s")
```

```console
$ python3 make-narration.py
sentence 1: 0.300 .. 1.815
sentence 2: 2.415 .. 7.195
sentence 3: 7.795 .. 9.635
sentence 4: 10.235 .. 11.695
total 12.195 s
$ cat script.txt
The cache was empty.
So every request went to the disk, and the disk was slow.
We added a warm-up step.
Now it starts fast.
```

So the right cuts are somewhere in 1.815–2.415, 7.195–7.795 and
9.635–10.235.

## Symptom 1: cuts from the character share land inside words

The duration, from the last `time=` of a decode to nowhere (there is no
`ffprobe` in the build I use):

```console
$ ffmpeg -hide_banner -i narration.wav -f null - 2>&1 | grep -o 'time=[^ ]*'
time=00:00:12.19
```

The guess: sentence *k* ends at duration × (characters up to and including
sentence *k*) / (all characters).

```python
#!/usr/bin/env python3
"""estimate.py DURATION < script.txt: sentence end times by character share."""
import sys
dur = float(sys.argv[1])
lines = [l.strip() for l in sys.stdin if l.strip()]
total = sum(len(l) for l in lines)
done = 0
for l in lines[:-1]:
    done += len(l)
    print(f"{dur * done / total:.3f}  after: {l}")
```

```console
$ python3 estimate.py 12.19 < script.txt
2.032  after: The cache was empty.
7.822  after: So every request went to the disk, and the disk was slow.
10.260  after: We added a warm-up step.
```

The first guess is inside the pause. The other two are not: 7.822 is 27 ms
after sentence 3 starts (7.795), and 10.260 is 25 ms after sentence 4 starts
(10.235). The guess ignores the leading silence and assumes every pause is
as long, per character, as speech. Cut sentence 3 there and look at the clip
with `silencedetect`:

```console
$ ffmpeg -hide_banner -loglevel error -y -i narration.wav -ss 7.822 -to 10.260 naive3.wav
$ ffmpeg -hide_banner -i naive3.wav -af silencedetect=noise=-50dB:d=0.02 -f null - 2>&1 | grep -E 'silence_|time='
[silencedetect @ 0x60000388c780] silence_start: 0.182833
[silencedetect @ 0x60000388c780] silence_end: 0.243208 | silence_duration: 0.060375
[silencedetect @ 0x60000388c780] silence_start: 0.617958
[silencedetect @ 0x60000388c780] silence_end: 0.678208 | silence_duration: 0.06025
[silencedetect @ 0x60000388c780] silence_start: 0.832875
[silencedetect @ 0x60000388c780] silence_end: 0.89325 | silence_duration: 0.060375
[silencedetect @ 0x60000388c780] silence_start: 1.377958
[silencedetect @ 0x60000388c780] silence_end: 1.43825 | silence_duration: 0.0602917
[silencedetect @ 0x60000388c780] silence_start: 1.812792
[silencedetect @ 0x60000388c780] silence_end: 2.413208 | silence_duration: 0.600417
size=N/A time=00:00:02.43 bitrate=N/A speed=2.82e+03x    
```

- There is no `silence_start: 0`: the clip begins with sound. "We" is 0.21 s
  long in the original, and here the first word ends at 0.183. The first
  27 ms of it are gone (they are at the end of the previous clip).
- After the 0.6 s pause (1.81 to 2.41) there is sound again until the end of
  the clip: the first 25 ms of "Now".

With a real voice that would likely be a clipped first consonant and a click of the next
word at the end. This generated "speech" has a perfectly even rate, and the
guesses are still off by about 25 ms. How far they are off on a real voice I
haven't measured.

## Symptom 2: `-ss` before `-i`, and the `-to` clip is too long

`-ss` before `-i` (an input option, sometimes recommended because ffmpeg then
seeks instead of decoding from the start; I didn't time it here) with `-to`
left as an output option:

```console
$ ffmpeg -hide_banner -loglevel error -y -ss 7.495 -i narration.wav -to 9.935 s3.wav; echo "exit $?"
exit 0
$ ffmpeg -hide_banner -i s3.wav -f null - 2>&1 | grep -o 'time=[^ ]*'
time=00:00:04.70
```

The clip should be 9.935 − 7.495 = 2.44 s. It's 4.70 s: everything from 7.495
to the end of the 12.19 s file. With `-ss` on the input, the output's
timestamps start at 0 again, so `-to 9.935` means "9.935 s into the clip",
which is past its end. Either of these gives 2.44 s:

```console
$ ffmpeg -hide_banner -loglevel error -y -ss 7.495 -i narration.wav -t 2.44 s3.wav
$ ffmpeg -hide_banner -i s3.wav -f null - 2>&1 | grep -o 'time=[^ ]*'
time=00:00:02.44
$ ffmpeg -hide_banner -loglevel error -y -ss 7.495 -to 9.935 -i narration.wav s3.wav
$ ffmpeg -hide_banner -i s3.wav -f null - 2>&1 | grep -o 'time=[^ ]*'
time=00:00:02.44
```

(`-t` is a length; `-to` *before* `-i` is a position in the input, like
`-ss` there.) Or put both after `-i`, as in the rest of this post (I didn't
compare the speed).

## Fix: move each guess into the longest pause near it

The character share is good enough to say *which* pause a sentence ends in,
not *where*. So: find the pauses with `silencedetect`, and for each guess
take the longest pause within a window around it. Cut in the middle of that
pause, so each clip keeps half of it as lead-in and tail.

Why the longest and not the nearest: with a short minimum duration,
`silencedetect` also reports the gaps between words. At `d=0.05` the pauses
around the second guess (7.822) are:

```console
$ ffmpeg -hide_banner -i narration.wav -af silencedetect=noise=-50dB:d=0.05 -f null - 2>&1 | grep -oE 'silence_(start|end): [0-9.]+' | paste - - | awk '$2 > 6.5 && $2 < 8.6'
silence_start: 6.759958	silence_end: 6.82025
silence_start: 7.194958	silence_end: 7.795292
silence_start: 8.004833	silence_end: 8.065208
silence_start: 8.439958	silence_end: 8.500208
$ ffmpeg -hide_banner -i narration.wav -af silencedetect=noise=-50dB:d=0.05 -f null - 2>&1 | grep -oE 'silence_(start|end): [0-9.]+' | paste - - | awk -v e=7.822 '{m=($2+$4)/2; d=m-e; if(d<0)d=-d; if(best==""||d<best){best=d;bm=m}} END{printf "nearest pause middle %.3f (%.3f s away)\n", bm, best}'
nearest pause middle 8.035 (0.213 s away)
```

The nearest middle is 8.035, the gap between "We" and "added". The sentence
pause (7.195 to 7.795, middle 7.495) is 0.327 s away. Snapping to the nearest
pause would put "We" at the end of sentence 2.

The script (`-50dB` is this file's threshold; for picking one, see the
previous post):

```python
#!/usr/bin/env python3
"""Split a narration into one clip per sentence, cutting inside pauses.

usage: split.py AUDIO SCRIPT [NOISE] [MINPAUSE] [WINDOW]
       e.g. split.py narration.wav script.txt -50dB 0.3 1.0
SCRIPT has one sentence per line. Writes clip_01.wav, clip_02.wav, ...
The ffmpeg binary is $FFMPEG, or "ffmpeg" on PATH.
"""
import os, re, subprocess, sys

audio, script = sys.argv[1], sys.argv[2]
noise = sys.argv[3] if len(sys.argv) > 3 else "-50dB"
minpause = float(sys.argv[4]) if len(sys.argv) > 4 else 0.3
window = float(sys.argv[5]) if len(sys.argv) > 5 else 1.0
ff = os.environ.get("FFMPEG", "ffmpeg")

lines = [l.strip() for l in open(script, encoding="utf-8") if l.strip()]

# silencedetect logs at "info" level on stderr: no -loglevel, read stderr.
r = subprocess.run([ff, "-hide_banner", "-i", audio, "-af",
                    f"silencedetect=noise={noise}:d={minpause}",
                    "-f", "null", "-"], stderr=subprocess.PIPE, text=True)
if r.returncode != 0:
    sys.exit(r.stderr.strip().splitlines()[-1])
m = re.findall(r"time=(\d+):(\d+):([\d.]+)", r.stderr)
if not m:
    sys.exit("could not read the duration from ffmpeg's output")
h, mi, s = m[-1]
dur = int(h) * 3600 + int(mi) * 60 + float(s)

pauses, start = [], None
for kind, t in re.findall(r"silence_(start|end): (-?[\d.]+)", r.stderr):
    if kind == "start":
        start = float(t)
    elif start is not None:
        pauses.append((start, float(t)))
        start = None
# pauses that touch the start or end of the file are not between sentences
inner = [(a, b) for a, b in pauses if a > 0.001 and b < dur - 0.001]

total = sum(len(l) for l in lines)
cuts, done = [], 0
for i, l in enumerate(lines[:-1]):
    done += len(l)
    est = dur * done / total
    near = [p for p in inner if p[1] >= est - window and p[0] <= est + window]
    if not near:
        sys.exit(f"no pause >= {minpause}s within {window}s of {est:.3f} "
                 f"(after line {i + 1}): nothing written")
    a, b = max(near, key=lambda p: (p[1] - p[0], -abs((p[0] + p[1]) / 2 - est)))
    cut = (a + b) / 2
    if cuts and cut <= cuts[-1]:
        sys.exit(f"lines {i} and {i + 1} picked the same pause at {cut:.3f}: "
                 "nothing written (check NOISE/MINPAUSE/WINDOW)")
    print(f"line {i + 1}: estimate {est:.3f}, pause {a:.3f}..{b:.3f}, cut {cut:.3f}")
    cuts.append(cut)

edges = [0.0] + cuts + [None]
for i in range(len(lines)):
    a, b = edges[i], edges[i + 1]
    out = f"clip_{i + 1:02d}.wav"
    # -ss and -to after -i: both are positions in the input file
    cmd = [ff, "-hide_banner", "-loglevel", "error", "-y", "-i", audio, "-ss", f"{a:.6f}"]
    if b is not None:
        cmd += ["-to", f"{b:.6f}"]
    if subprocess.run(cmd + [out]).returncode != 0:
        sys.exit(f"ffmpeg failed on {out}")
    print(f"{out}: {a:.3f} .. {'end' if b is None else f'{b:.3f}'}  {lines[i]}")
```

What it does:

- One `silencedetect` run with no `-loglevel` (its results are `info` lines
  on stderr), which also gives the duration from the final `time=`.
- Pauses that touch the start or the end of the file are dropped; they are
  lead-in and tail, not sentence breaks.
- For each guess: every pause that overlaps guess ± `WINDOW`, the longest
  of them (on a tie, the one closer to the guess), cut at its middle.
- All cut points are chosen before anything is written. If a guess has no
  pause in its window, or two sentences pick the same pause, it stops with
  exit 1 and writes no clips.

Saved as `split.py`:

```console
$ python3 split.py narration.wav script.txt; echo "exit $?"
line 1: estimate 2.032, pause 1.815..2.415, cut 2.115
line 2: estimate 7.822, pause 7.195..7.795, cut 7.495
line 3: estimate 10.260, pause 9.635..10.235, cut 9.935
clip_01.wav: 0.000 .. 2.115  The cache was empty.
clip_02.wav: 2.115 .. 7.495  So every request went to the disk, and the disk was slow.
clip_03.wav: 7.495 .. 9.935  We added a warm-up step.
clip_04.wav: 9.935 .. end  Now it starts fast.
exit 0
```

Check every clip: it should start and end with silence, and the sound in
between should be as long as the sentence in the original.

```console
$ for f in clip_0*.wav; do printf '%s: ' $f; ffmpeg -hide_banner -i $f -af silencedetect=noise=-50dB:d=0.1 -f null - 2>&1 | grep -oE 'silence_(start|end): [0-9.]+|time=[0-9:.]+' | tr '\n' ' '; echo; done
clip_01.wav: silence_start: 0 silence_end: 0.300167 silence_start: 1.814958 silence_end: 2.115125 time=00:00:02.11 
clip_02.wav: silence_start: 0 silence_end: 0.300167 silence_start: 2.899875 silence_end: 3.350083 silence_start: 5.079833 silence_end: 5.38 time=00:00:05.38 
clip_03.wav: silence_start: 0 silence_end: 0.300167 silence_start: 2.139667 silence_end: 2.439875 time=00:00:02.43 
clip_04.wav: silence_start: 0 silence_end: 0.300208 silence_start: 1.759875 silence_end: 2.26 time=00:00:02.26 
```

- Every clip starts with 0.3 s of silence and ends with 0.3 s (the last one
  0.5 s, the file's own tail).
- Clip 3: sound from 0.30 to 2.14, 1.84 s, the same as 7.795 to 9.635 in the
  original. "We" is whole again.
- Clip 2 has a 0.45 s pause in the middle: the comma. With `d=0.1` it shows
  up here, but it was never picked as a sentence break, because the window
  around each guess contained a longer pause.

With word gaps included (`MINPAUSE` 0.05), the longest-pause rule still
picks the same cuts:

```console
$ python3 split.py narration.wav script.txt -50dB 0.05 | head -3
line 1: estimate 2.032, pause 1.815..2.415, cut 2.115
line 2: estimate 7.822, pause 7.195..7.795, cut 7.495
line 3: estimate 10.260, pause 9.635..10.235, cut 9.935
```

When it can't find a pause, it stops before writing anything. A threshold
below this file's noise floor finds no pauses at all:

```console
$ rm clip_0*.wav
$ python3 split.py narration.wav script.txt -60dB; echo "exit $?"
no pause >= 0.3s within 1.0s of 2.032 (after line 1): nothing written
exit 1
$ ls clip_01.wav
ls: clip_01.wav: No such file or directory
```

A window too small for the guess's error:

```console
$ python3 split.py narration.wav script.txt -50dB 0.3 0.01; echo "exit $?"
line 1: estimate 2.032, pause 1.815..2.415, cut 2.115
no pause >= 0.3s within 0.01s of 7.822 (after line 2): nothing written
exit 1
$ ls clip_01.wav
ls: clip_01.wav: No such file or directory
```

The window is the one setting that depends on the text: it has to be larger
than how far the character share can be off, and smaller than the distance
to the next sentence break. The same-pause check is there for a window that
is too large; it didn't trigger on this file, and I have only tried the
script on this file.

## Checklist

- A cut time computed from character counts tells you which pause a
  sentence ends in, not where. Cut inside a detected pause, not at the guess.
- Check each clip with `silencedetect`: no `silence_start: 0` means it starts
  in the middle of sound; sound right at the end means it caught the next
  word.
- Snap to the longest pause near the guess, not the nearest: with a short
  `d`, gaps between words count as pauses too.
- Leave out pauses that touch the start or end of the file.
- `-ss` before `-i` resets the clip's time to 0: an output `-to` is then
  measured from the new start, so the clip is too long (here it ran to the
  end of the file), with exit 0. Use `-t LENGTH`, put `-to` before `-i` too, or put both after `-i`.
- `silencedetect` results are `info` lines on stderr: don't run the
  detection with `-loglevel error`.
- The `0x…` filter addresses and the `speed=` values differ between runs.

*Tested on macOS 26.5.2 (Apple silicon) with ffmpeg 7.1, the static arm64
build shipped in the imageio-ffmpeg Python package (version line above), and
/usr/bin/python3 3.8.2, from zsh. The narration is a generated 24 kHz mono
WAV of tones with a noise floor, not real TTS output, with an even speaking
rate. No real voices, compressed input (MP3/AAC), `-c copy`, stereo files
or other ffmpeg builds tested.*
