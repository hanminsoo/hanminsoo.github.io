---
title: "ffmpeg silencedetect prints nothing, or cuts off word endings: shortening TTS pauses without clipping words"
description: >-
  ffmpeg's silencedetect printed no silence at all with its defaults (2 s
  minimum, -60 dB, below the clip's noise floor) and with -loglevel warning,
  and ffmpeg still exited 0. At -30 dB it put the pause 0.12 s early and 0.2 s
  late, so cutting at its timestamps ate a decaying word ending and a quiet
  onset. Measure the floor with astats, sweep the threshold, and cut only the
  middle of each pause. on this clip silenceremove left a longer pause than stop_silence.
  ffmpeg 7.1.
tags: [ffmpeg, audio, tts, macos]
---

You want to tighten the pauses in a text-to-speech narration: every gap
longer than half a second should become half a second. `silencedetect`
should tell you where the gaps are. The first run prints **no
`silence_start` line at all**, and ffmpeg exits 0. After raising the
threshold you get timestamps, but cutting at them **eats the end of one word
and the start of the next**, with no error either.

Below I reproduce both with a generated clip whose boundaries I know, show how
I picked a threshold, and give a small script that only cuts the middle of
each pause. At the end, what `silenceremove` did with the same clip.

The ffmpeg I used:

```console
$ ffmpeg -version | head -1
ffmpeg version 7.1 Copyright (c) 2000-2024 the FFmpeg developers
```

Everything below runs in one empty folder, in this order.

## A test clip with known word boundaries

Real TTS output would do, but then I can't say where a word "really" ends.
So this script builds a 5-second, 24 kHz mono clip from tones, with a noise
floor under everything:

```sh
# make-clip.sh: a TTS-like test clip with known boundaries
#  0.3-0.9  word 1, then a tail that fades out until 1.3 (still -41 dB at 1.3)
#  1.3-2.3  pause (1.0 s)
#  2.3-2.5  quiet onset at -34 dB (like a soft consonant), then word 2 to 3.1
#  3.1-3.5  pause (0.4 s, short: should stay)
#  3.5-4.0  word 3, then 1.0 s of pause to the end
#  noise floor: uniform noise, peaks at -54 dB
E='0.5*sin(2*PI*220*t)*between(t,0.3,0.9)'
E="$E"'+0.5*exp(-10*(t-0.9))*sin(2*PI*220*t)*gt(t,0.9)*lt(t,1.3)'
E="$E"'+0.02*sin(2*PI*330*t)*between(t,2.3,2.5)'
E="$E"'+0.5*sin(2*PI*220*t)*gt(t,2.5)*lt(t,3.1)'
E="$E"'+0.5*sin(2*PI*260*t)*between(t,3.5,4.0)'
E="$E"'+0.002*(2*random(0)-1)'
ffmpeg -hide_banner -loglevel error -y -f lavfi -i "aevalsrc='$E':s=24000:d=5" -c:a pcm_s16le tts.wav
```

The single quotes inside `aevalsrc='…'` matter: without them ffmpeg splits
the expression at its commas and fails with `No such filter: '0.3'`.

```console
$ sh make-clip.sh
$ stat -f '%N %z bytes' tts.wav
tts.wav 240078 bytes
```

So the right answer is: sound until 1.3, pause until 2.3, sound until 3.1,
pause until 3.5, and so on.

## Symptom 1: no output with the defaults

```console
$ ffmpeg -hide_banner -i tts.wav -af silencedetect -f null - 2>sd.log; echo "exit $?"
exit 0
$ grep -c silence_ sd.log
0
```

Nothing, and exit 0. The defaults explain it:

```console
$ ffmpeg -hide_banner -h filter=silencedetect 2>&1 | grep -E '^ +(noise|d) '
   noise             <double>     ..F.A...... set noise tolerance (from 0 to DBL_MAX) (default 0.001)
   d                 <duration>   ..F.A...... set minimum duration in seconds (default 2)
```

- `d=2`: only silences of **2 seconds or more** count. TTS pauses are
  shorter.
- `noise=0.001` is -60 dB. A sample counts as silent only below that, and this
  clip's floor peaks at -54 dB, so even a short `d` finds nothing:

```console
$ ffmpeg -hide_banner -i tts.wav -af silencedetect=d=0.3 -f null - 2>&1 | grep -c silence_
0
```

### Also no output: `-loglevel` and `2>/dev/null`

`silencedetect` writes its results as log lines on **stderr**, at the
`info` level. With a quieter log level, or stderr thrown away, they're gone
too, even with a threshold that works:

```console
$ ffmpeg -hide_banner -loglevel warning -i tts.wav -af silencedetect=noise=-50dB:d=0.3 -f null - 2>&1 | grep -c silence_
0
$ ffmpeg -hide_banner -i tts.wav -af silencedetect=noise=-50dB:d=0.3 -f null - 2>/dev/null | grep -c silence_
0
$ ffmpeg -hide_banner -i tts.wav -af silencedetect=noise=-50dB:d=0.3 -f null - 2>&1 | grep -c silence_
8
```

So a script that runs ffmpeg with `-loglevel error` "to keep it quiet" sees no
pauses and happily does nothing.

## Symptom 2: a high threshold moves the boundaries into the words

`-30dB` is a common first guess. It finds the pauses, but look where:

```console
$ ffmpeg -hide_banner -i tts.wav -af silencedetect=noise=-30dB:d=0.3 -f null - 2>&1 | grep silence_
[silencedetect @ 0x600001a246c0] silence_start: 0
[silencedetect @ 0x600001a246c0] silence_end: 0.300083 | silence_duration: 0.300083
[silencedetect @ 0x600001a246c0] silence_start: 1.180792
[silencedetect @ 0x600001a246c0] silence_end: 2.500083 | silence_duration: 1.319292
[silencedetect @ 0x600001a246c0] silence_start: 3.099958
[silencedetect @ 0x600001a246c0] silence_end: 3.500042 | silence_duration: 0.400083
[silencedetect @ 0x600001a246c0] silence_start: 4
[silencedetect @ 0x600001a246c0] silence_end: 5 | silence_duration: 1
```

The pause after word 1 "starts" at 1.18, but the tail goes on to 1.3. It
"ends" at 2.5, but the quiet onset starts at 2.3. Everything quieter than
-30 dB is treated as pause. (The filter address `0x…` differs between runs.)

Cut out exactly that span, 1.180792 to 2.500083, and check the result with
a threshold that works:

```console
$ ffmpeg -hide_banner -loglevel error -y -i tts.wav -filter_complex "[0:a]asplit[a][b];[a]atrim=end=1.180792[x];[b]atrim=start=2.500083,asetpts=PTS-STARTPTS[y];[x][y]concat=n=2:v=0:a=1" naive.wav
$ ffmpeg -hide_banner -i naive.wav -af silencedetect=noise=-50dB:d=0.1 -f null - 2>&1 | grep silence_
[silencedetect @ 0x600003e24000] silence_start: 0
[silencedetect @ 0x600003e24000] silence_end: 0.300042 | silence_duration: 0.300042
[silencedetect @ 0x600003e24000] silence_start: 1.780708
[silencedetect @ 0x600003e24000] silence_end: 2.18075 | silence_duration: 0.400042
[silencedetect @ 0x600003e24000] silence_start: 2.680708
[silencedetect @ 0x600003e24000] silence_end: 3.680708 | silence_duration: 1
```

Words 1 and 2 are now one block of sound from 0.3 to 1.78: 1.48 s, where the
two words were 1.0 s + 0.8 s. The missing 0.32 s is the tail (0.12 s) and the
onset (0.2 s). There's no gap left between the words at all, and the cut
lands in the tail's waveform while it is still around -30 dB, not in the
noise floor.

In real speech those quiet parts are the ends of words and soft consonants,
so this is the "it clips my words" result.

## Pick the threshold from the noise floor

Measure the level inside a stretch you know is a pause (here 1.5 to 2.0 s)
with `astats`:

```console
$ ffmpeg -hide_banner -i tts.wav -af atrim=1.5:2.0,astats -f null - 2>&1 | grep -E 'Peak level|RMS level' | head -2
[Parsed_astats_1 @ 0x60000046c790] Peak level dB: -53.917855
[Parsed_astats_1 @ 0x60000046c790] RMS level dB: -58.712387
```

The floor peaks at -54 dB, so the threshold has to be **above** that (or no
sample run is "silent") but as close to it as you can get. A sweep shows
where the boundaries stop moving; this prints the pause after word 1:

```console
$ for n in -30 -40 -50 -60; do printf '%s dB: ' $n; ffmpeg -hide_banner -i tts.wav -af silencedetect=noise=${n}dB:d=0.3 -f null - 2>&1 | grep -oE 'silence_(start|end): [0-9.]+' | sed -n '3,4p' | tr '\n' ' '; echo; done
-30 dB: silence_start: 1.180792 silence_end: 2.500083 
-40 dB: silence_start: 1.299208 silence_end: 2.300292 
-50 dB: silence_start: 1.299792 silence_end: 2.300125 
-60 dB: 
```

From -30 to -40 the boundaries jump to the true 1.3 and 2.3. From -40 to -50
they barely move. At -60 (below the floor) nothing is found. Here -50 dB is
the pick. On a different voice or file, run the same two commands; the
numbers will be different.

## Fix: cut only the middle of each pause

Even with a good threshold, don't cut at the reported edges. Keep half of
the target pause on each side, next to the words, and remove only what's in
between. Then a word ending that is a bit quieter than the threshold still
stays.

```python
#!/usr/bin/env python3
"""Shorten every pause longer than MAX seconds to MAX seconds.

usage: shorten.py IN OUT [NOISE] [MAX]      e.g. shorten.py in.wav out.wav -50dB 0.5
The ffmpeg binary is $FFMPEG, or "ffmpeg" on PATH.
"""
import os, re, subprocess, sys

src, dst = sys.argv[1], sys.argv[2]
noise = sys.argv[3] if len(sys.argv) > 3 else "-50dB"
maxp = float(sys.argv[4]) if len(sys.argv) > 4 else 0.5
ff = os.environ.get("FFMPEG", "ffmpeg")
pad = maxp / 2  # kept on each side of a pause, next to the words

# silencedetect logs at "info" level on stderr: no -loglevel, read stderr.
r = subprocess.run([ff, "-hide_banner", "-nostats", "-i", src,
                    "-af", f"silencedetect=noise={noise}:d={maxp}",
                    "-f", "null", "-"],
                   stderr=subprocess.PIPE, text=True)
if r.returncode != 0:
    sys.exit(r.stderr.strip().splitlines()[-1])

pauses, start = [], None
for m in re.finditer(r"silence_(start|end): (-?[0-9.]+)", r.stderr):
    t = float(m.group(2))
    if m.group(1) == "start":
        start = t
    elif start is not None:
        pauses.append((start, t))
        start = None
if start is not None:              # file ends inside a pause, no silence_end
    pauses.append((start, None))
if not pauses:
    sys.exit(f"no pause longer than {maxp}s below {noise}: "
             "nothing written (check NOISE against the file's noise floor)")

# keep: from 0 to the first cut, between cuts, after the last cut
keep, pos = [], 0.0
for s, e in pauses:
    keep.append((pos, s + pad))
    pos = None if e is None else e - pad
    if pos is None:
        break
if pos is not None:
    keep.append((pos, None))

n = len(keep)
parts = [f"[0:a]asplit={n}" + "".join(f"[s{i}]" for i in range(n))]
for i, (a, b) in enumerate(keep):
    end = "" if b is None else f":end={b:.6f}"
    parts.append(f"[s{i}]atrim=start={a:.6f}{end},asetpts=PTS-STARTPTS[k{i}]")
parts.append("".join(f"[k{i}]" for i in range(n)) + f"concat=n={n}:v=0:a=1[out]")

r = subprocess.run([ff, "-hide_banner", "-loglevel", "error", "-y", "-i", src,
                    "-filter_complex", ";".join(parts), "-map", "[out]", dst])
if r.returncode != 0:
    sys.exit(r.returncode)
print(f"{len(pauses)} pauses shortened to {maxp}s -> {dst}")
for a, b in keep:
    print(f"  kept {a:.3f} .. {'end' if b is None else f'{b:.3f}'}")
```

What it does:

- `d={maxp}` makes `silencedetect` report only pauses longer than the
  target, so the 0.4 s pause and the 0.3 s lead-in are left alone.
- No `-loglevel` on the detection run (and `-nostats` only drops the progress
  line), so the `silence_` lines are there to parse. If none are found it
  stops with exit 1 instead of writing an unchanged copy.
- `atrim` + `asetpts` + `concat` joins the kept pieces.

Saved as `shorten.py`:

```console
$ python3 shorten.py tts.wav out.wav -50dB 0.5; echo "exit $?"
2 pauses shortened to 0.5s -> out.wav
  kept 0.000 .. 1.550
  kept 2.050 .. 4.250
  kept 4.750 .. end
exit 0
$ ffmpeg -hide_banner -i out.wav -af silencedetect=noise=-50dB:d=0.1 -f null - 2>&1 | grep -E 'silence_|time='
[silencedetect @ 0x6000037d86c0] silence_start: 0
[silencedetect @ 0x6000037d86c0] silence_end: 0.300042 | silence_duration: 0.300042
[silencedetect @ 0x6000037d86c0] silence_start: 1.299792
[silencedetect @ 0x6000037d86c0] silence_end: 1.799792 | silence_duration: 0.5
[silencedetect @ 0x6000037d86c0] silence_start: 2.599667
[silencedetect @ 0x6000037d86c0] silence_end: 2.999708 | silence_duration: 0.400042
[silencedetect @ 0x6000037d86c0] silence_start: 3.499667
[silencedetect @ 0x6000037d86c0] silence_end: 3.999667 | silence_duration: 0.5
size=N/A time=00:00:03.99 bitrate=N/A speed=6.02e+03x    
```

Check against the original:

- Word 1 still ends at 1.3. The 1.0 s pause is now 0.5 s.
- Word 2 runs 1.80 to 2.60: 0.8 s, the same as 2.3 to 3.1 in the original,
  so the quiet onset is still there.
- The 0.4 s pause is unchanged; the 1.0 s at the end became 0.5 s.
  5.0 s became 4.0 s.

With the floor threshold too low, it stops instead of writing:

```console
$ python3 shorten.py tts.wav out60.wav -60dB 0.5; echo "exit $?"
no pause longer than 0.5s below -60dB: nothing written (check NOISE against the file's noise floor)
exit 1
$ ls out60.wav
ls: out60.wav: No such file or directory
```

The padding also softens a bad threshold. At -30 dB the words survived on
this clip, because 0.25 s of padding covers the 0.12 s tail and the 0.2 s
onset, but the pause came out at 0.18 s instead of 0.5 s:

```console
$ python3 shorten.py tts.wav out30.wav -30dB 0.5
2 pauses shortened to 0.5s -> out30.wav
  kept 0.000 .. 1.431
  kept 2.250 .. 4.250
  kept 4.750 .. end
$ ffmpeg -hide_banner -i out30.wav -af silencedetect=noise=-50dB:d=0.1 -f null - 2>&1 | grep silence_ | sed -n '3,4p'
[silencedetect @ 0x600001f240c0] silence_start: 1.299792
[silencedetect @ 0x600001f240c0] silence_end: 1.480833 | silence_duration: 0.181042
```

With a smaller padding the words are cut again. A 0.2 s target means 0.1 s
of padding, less than the 0.2 s onset:

```console
$ python3 shorten.py tts.wav out30b.wav -30dB 0.2
4 pauses shortened to 0.2s -> out30b.wav
  kept 0.000 .. 0.100
  kept 0.200 .. 1.281
  kept 2.400 .. 3.200
  kept 3.400 .. 4.100
  kept 4.900 .. end
```

It keeps word 1 only up to 1.281 (the tail runs to 1.3) and resumes at 2.4,
so the first 0.1 s of the onset (2.3 to 2.4) is gone. Measuring the floor is
still the real fix.

## What about `silenceremove`?

`silenceremove` can do this in one filter, with the same threshold
problem. With `stop_periods=-1` (every pause) and `stop_silence` (how much to
keep):

```console
$ ffmpeg -hide_banner -loglevel error -y -i tts.wav -af "silenceremove=stop_periods=-1:stop_duration=0.5:stop_threshold=-30dB:stop_silence=0.25" sr30.wav
$ ffmpeg -hide_banner -i sr30.wav -af silencedetect=noise=-50dB:d=0.1 -f null - 2>&1 | grep -E 'silence_|time='
[silencedetect @ 0x600002a50000] silence_start: 0
[silencedetect @ 0x600002a50000] silence_end: 0.300042 | silence_duration: 0.300042
[silencedetect @ 0x600002a50000] silence_start: 1.299792
[silencedetect @ 0x600002a50000] silence_end: 1.902375 | silence_duration: 0.602583
[silencedetect @ 0x600002a50000] silence_start: 2.501875
[silencedetect @ 0x600002a50000] silence_end: 2.901917 | silence_duration: 0.400042
[silencedetect @ 0x600002a50000] silence_start: 3.401875
[silencedetect @ 0x600002a50000] silence_end: 4.171417 | silence_duration: 0.769542
size=N/A time=00:00:04.17 bitrate=N/A speed=5.94e+03x    
```

Word 2 is now 1.90 to 2.50, 0.6 s instead of 0.8 s: at -30 dB the quiet
onset was removed. The tail of word 1 survived (sound runs to 1.3).

Two more things I saw, without reading the filter's source:

- The pauses left over were 0.60 s and 0.77 s, not the 0.25 s I asked to
  keep, nor the 0.5 s `stop_duration`.
- At -50 dB, how much pause was left depended on both options. The second
  number below is the pause after word 1 (the true pause is 1.0 s):

```console
$ for p in 0.5:0 0.3:0 0.3:0.2 0.5:0.25; do d=${p%%:*}; s=${p#*:}; ffmpeg -hide_banner -loglevel error -y -i tts.wav -af "silenceremove=stop_periods=-1:stop_duration=${d}:stop_threshold=-50dB:stop_silence=${s}" x.wav; printf 'stop_duration=%s stop_silence=%s: ' $d $s; ffmpeg -hide_banner -i x.wav -af silencedetect=noise=-50dB:d=0.1 -f null - 2>&1 | grep silence_duration | sed -n 2p | sed 's/.*silence_duration: //'; done
stop_duration=0.5 stop_silence=0: 0.516625
stop_duration=0.3 stop_silence=0: 0.316625
stop_duration=0.3 stop_silence=0.2: 0.516625
stop_duration=0.5 stop_silence=0.25: 0.766625
```

On this clip the pause left was about `stop_duration + stop_silence`, plus
about 17 ms. If you use `silenceremove`, measure the result the same way rather
than trusting the option names.

Write `${d}` and `${s}` with braces in zsh. Unbraced, zsh reads the `:s`
after `$d` as a modifier on the variable, even inside double quotes:

```console
$ d=0.5; s=0; print -r -- "a=$d:stop_threshold=-50dB:stop_silence=$s:window=0.02"
a=0.5op_silence=0:window=0.02
$ print -r -- "a=$d:stop_silence=0"
zsh: bad substitution
```

The first one is what ffmpeg then rejects with `Unable to parse option value
"0.5op_silence=0" as duration`. bash prints these strings unchanged.

## Checklist

- No `silence_start` at all: the defaults are `d=2` (2 s) and `noise=-60dB`.
  Set both, e.g. `silencedetect=noise=-50dB:d=0.3`.
- The results are log lines on stderr at `info` level. `-loglevel warning`
  or `2>/dev/null` hides them, and ffmpeg still exits 0.
- Measure the noise floor in a known pause: `-af atrim=A:B,astats`, read
  `Peak level dB`. The threshold must be above it, or nothing counts as
  silence.
- Sweep the threshold and pick the level where `silence_start` and
  `silence_end` stop moving. Too high (-30 dB here) puts them inside the
  quiet ends and starts of words.
- Don't cut at the reported edges. Keep part of the pause on both sides and
  remove only the middle (`atrim` pieces + `concat`).
- Check the output with `silencedetect` again: the sound between pauses
  should be as long as before.
- `silenceremove`: same threshold problem; on this clip the pause it left
  was longer than `stop_silence`.
- The `0x…` filter addresses and the `speed=` values differ between runs.

*Tested on macOS 26.5.2 (Apple silicon) with ffmpeg 7.1, the static arm64
build shipped in the imageio-ffmpeg Python package (version line above), and
/usr/bin/python3 3.8.2, from zsh. The clip is a generated 24 kHz mono WAV of
tones with a noise floor, not real TTS output; on a real voice, measure the
floor and sweep the threshold as shown. No other ffmpeg builds, compressed
input (MP3/AAC) or stereo files tested.*
