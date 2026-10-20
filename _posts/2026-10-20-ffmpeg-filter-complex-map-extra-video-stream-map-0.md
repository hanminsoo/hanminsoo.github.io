---
title: "ffmpeg -filter_complex with -map \"[v]\" -map 0: the output has two video streams and the default one is the unfiltered original"
description: >-
  To keep the audio after a -filter_complex, adding -map 0 next to -map "[v]"
  also copied the original, unfiltered video into the mp4: three streams,
  with the (default) flag on the original. -map 0 means every stream of input
  0. Map only the audio (-map 0:a, or -map "0:a?" when it may be missing).
  Also: a negative map placed after "[v]" breaks the filter, and a failed run
  can leave the previous out.mp4 — or a 0-byte file — behind. ffmpeg 7.1.
tags: [ffmpeg, video, macos]
---

You scale, crop or overlay a video with `-filter_complex`, label the result
`[v]`, and `-map "[v]"`. The audio disappears, so you add `-map 0` to bring it
back. The exit code is 0 and nothing is printed, but the file has **two video
streams**, the original and yours, and the `(default)` flag sits on the
original.

Below I reproduce it with a generated test clip, explain why `-map 0` does
this, and show the fixes and two related traps I hit while testing.

The ffmpeg I used:

```console
$ ffmpeg -version | head -1
ffmpeg version 7.1 Copyright (c) 2000-2024 the FFmpeg developers
```

Everything below runs in one empty folder, in this order.

## A test clip and a way to list streams

Two seconds of test pattern (320x240) with a 440 Hz tone, plus the same video
without audio for later:

```console
$ ffmpeg -hide_banner -loglevel error -f lavfi -i testsrc=s=320x240:d=2:r=25 -f lavfi -i sine=f=440:d=2 -c:v libx264 -c:a aac -shortest in.mp4
$ ffmpeg -hide_banner -loglevel error -f lavfi -i testsrc=s=320x240:d=2:r=25 -c:v libx264 noaudio.mp4
```

This lists the streams with `ffmpeg -i`, so it works without `ffprobe`; the
`grep` keeps only the stream lines:

```sh
# streams.sh — list the streams in a file (no ffprobe needed)
ffmpeg -hide_banner -i "$1" 2>&1 | grep 'Stream #'
```

```console
$ sh streams.sh in.mp4
  Stream #0:0[0x1](und): Video: h264 (High 4:4:4 Predictive) (avc1 / 0x31637661), yuv444p(progressive), 320x240 [SAR 1:1 DAR 4:3], 47 kb/s, 25 fps, 25 tbr, 12800 tbn (default)
  Stream #0:1[0x2](und): Audio: aac (LC) (mp4a / 0x6134706D), 44100 Hz, mono, fltp, 69 kb/s (default)
```

One video, one audio.

## Reproduce: `-map "[v]" -map 0`

The filter halves the size to 160x120, so it's easy to tell which stream is
which:

```console
$ ffmpeg -hide_banner -loglevel error -i in.mp4 -filter_complex "[0:v]scale=160:120[v]" -map "[v]" -map 0 1.mp4; echo "exit $?"
exit 0
$ sh streams.sh 1.mp4
  Stream #0:0[0x1](und): Video: h264 (High 4:4:4 Predictive) (avc1 / 0x31637661), yuv444p(progressive), 160x120 [SAR 1:1 DAR 4:3], 32 kb/s, 25 fps, 25 tbr, 12800 tbn
  Stream #0:1[0x2](und): Video: h264 (High 4:4:4 Predictive) (avc1 / 0x31637661), yuv444p(progressive), 320x240 [SAR 1:1 DAR 4:3], 46 kb/s, 25 fps, 25 tbr, 12800 tbn (default)
  Stream #0:2[0x3](und): Audio: aac (LC) (mp4a / 0x6134706D), 44100 Hz, mono, fltp, 69 kb/s (default)
```

Three streams: the filtered 160x120 video, the **original 320x240 video**,
and the audio. The `(default)` flag is on the 320x240 one, not
on the filtered one.

With the maps the other way round, the original comes first:

```console
$ ffmpeg -hide_banner -loglevel error -i in.mp4 -filter_complex "[0:v]scale=160:120[v]" -map 0 -map "[v]" 2.mp4; echo "exit $?"
exit 0
$ sh streams.sh 2.mp4
  Stream #0:0[0x1](und): Video: h264 (High 4:4:4 Predictive) (avc1 / 0x31637661), yuv444p(progressive), 320x240 [SAR 1:1 DAR 4:3], 46 kb/s, 25 fps, 25 tbr, 12800 tbn (default)
  Stream #0:1[0x2](und): Audio: aac (LC) (mp4a / 0x6134706D), 44100 Hz, mono, fltp, 69 kb/s (default)
  Stream #0:2[0x3](und): Video: h264 (High 4:4:4 Predictive) (avc1 / 0x31637661), yuv444p(progressive), 160x120 [SAR 1:1 DAR 4:3], 32 kb/s, 25 fps, 25 tbr, 12800 tbn
```

Now the unfiltered video is stream 0 and carries the `(default)` flag.

## Why: `-map 0` means every stream of input 0

With a `-map`, ffmpeg takes only what you mapped, which is why `-map "[v]"`
alone drops the audio:

```console
$ ffmpeg -hide_banner -loglevel error -i in.mp4 -filter_complex "[0:v]scale=160:120[v]" -map "[v]" 3.mp4; echo "exit $?"
exit 0
$ sh streams.sh 3.mp4
  Stream #0:0[0x1](und): Video: h264 (High 4:4:4 Predictive) (avc1 / 0x31637661), yuv444p(progressive), 160x120 [SAR 1:1 DAR 4:3], 32 kb/s, 25 fps, 25 tbr, 12800 tbn (default)
```

`-map 0` then adds **all** of input 0: its audio, and also its video. The
filter reading `[0:v]` doesn't take that stream out of the list. So the
original video goes into the output a second time, next to the filtered one.

Why the `(default)` flag ends up on the original rather than the filtered
stream in `1.mp4` I didn't check; I only see that it does.

## Fix: map only the audio

```console
$ ffmpeg -hide_banner -loglevel error -i in.mp4 -filter_complex "[0:v]scale=160:120[v]" -map "[v]" -map 0:a 4.mp4; echo "exit $?"
exit 0
$ sh streams.sh 4.mp4
  Stream #0:0[0x1](und): Video: h264 (High 4:4:4 Predictive) (avc1 / 0x31637661), yuv444p(progressive), 160x120 [SAR 1:1 DAR 4:3], 32 kb/s, 25 fps, 25 tbr, 12800 tbn (default)
  Stream #0:1[0x2](und): Audio: aac (LC) (mp4a / 0x6134706D), 44100 Hz, mono, fltp, 69 kb/s (default)
```

One filtered video, one audio, and the default flag on the filtered video.

**If some inputs have no audio**, `-map 0:a` is an error:

```console
$ ffmpeg -hide_banner -loglevel error -i noaudio.mp4 -filter_complex "[0:v]scale=160:120[v]" -map "[v]" -map 0:a 5.mp4; echo "exit $?"
Stream map '' matches no streams.
To ignore this, add a trailing '?' to the map.
Failed to set value '0:a' for option 'map': Invalid argument
Error parsing options for output file 5.mp4.
Error opening output files: Invalid argument
exit 234
```

(The empty `''` in the first line is what this ffmpeg build actually printed.)


Add the `?` the message suggests. In zsh, quote it, or zsh treats `?` as a
wildcard and never runs ffmpeg:

```console
$ ffmpeg -hide_banner -loglevel error -i noaudio.mp4 -filter_complex "[0:v]scale=160:120[v]" -map "[v]" -map 0:a? 9.mp4; echo "exit $?"
zsh:1: no matches found: 0:a?
exit 1
```

Quoted:

```console
$ ffmpeg -hide_banner -loglevel error -i noaudio.mp4 -filter_complex "[0:v]scale=160:120[v]" -map "[v]" -map "0:a?" 6.mp4; echo "exit $?"
exit 0
$ sh streams.sh 6.mp4
  Stream #0:0[0x1](und): Video: h264 (High 4:4:4 Predictive) (avc1 / 0x31637661), yuv444p(progressive), 160x120 [SAR 1:1 DAR 4:3], 32 kb/s, 25 fps, 25 tbr, 12800 tbn (default)
```

## A negative map works only before the label

You can also keep `-map 0` and remove its video with a negative map,
`-map -0:v`. Placed after `-map "[v]"`, it broke the command:

```console
$ ffmpeg -hide_banner -loglevel error -i in.mp4 -filter_complex "[0:v]scale=160:120[v]" -map "[v]" -map 0 -map -0:v 7.mp4; echo "exit $?"
[fc#0 @ 0x60000336c0a0] Filter scale:default has an unconnected output
Error binding filtergraph inputs/outputs: Invalid argument
exit 234
```

The filter's output ended up unconnected, as if the negative map had also
removed the `[v]` mapping; I didn't dig further. Placed before `-map "[v]"`,
it works, but the audio becomes stream 0:

```console
$ ffmpeg -hide_banner -loglevel error -i in.mp4 -filter_complex "[0:v]scale=160:120[v]" -map 0 -map -0:v -map "[v]" 8.mp4; echo "exit $?"
exit 0
$ sh streams.sh 8.mp4
  Stream #0:0[0x1](und): Audio: aac (LC) (mp4a / 0x6134706D), 44100 Hz, mono, fltp, 69 kb/s (default)
  Stream #0:1[0x2](und): Video: h264 (High 4:4:4 Predictive) (avc1 / 0x31637661), yuv444p(progressive), 160x120 [SAR 1:1 DAR 4:3], 32 kb/s, 25 fps, 25 tbr, 12800 tbn (default)
```

`-map "[v]" -map 0:a` is shorter and keeps the video first.

## Trap while testing: a failed run leaves the old file

If you check your fix by overwriting the same `out.mp4`, a failed run makes
it look like the fix didn't work. First the broken command, then a "fixed"
one that fails because the input has no audio:

```console
$ ffmpeg -hide_banner -loglevel error -i in.mp4 -filter_complex "[0:v]scale=160:120[v]" -map "[v]" -map 0 -y out.mp4; echo "exit $?"
exit 0
$ ffmpeg -hide_banner -loglevel error -i noaudio.mp4 -filter_complex "[0:v]scale=160:120[v]" -map "[v]" -map 0:a -y out.mp4; echo "exit $?"
Stream map '' matches no streams.
To ignore this, add a trailing '?' to the map.
Failed to set value '0:a' for option 'map': Invalid argument
Error parsing options for output file out.mp4.
Error opening output files: Invalid argument
exit 234
$ sh streams.sh out.mp4
  Stream #0:0[0x1](und): Video: h264 (High 4:4:4 Predictive) (avc1 / 0x31637661), yuv444p(progressive), 160x120 [SAR 1:1 DAR 4:3], 32 kb/s, 25 fps, 25 tbr, 12800 tbn
  Stream #0:1[0x2](und): Video: h264 (High 4:4:4 Predictive) (avc1 / 0x31637661), yuv444p(progressive), 320x240 [SAR 1:1 DAR 4:3], 46 kb/s, 25 fps, 25 tbr, 12800 tbn (default)
  Stream #0:2[0x3](und): Audio: aac (LC) (mp4a / 0x6134706D), 44100 Hz, mono, fltp, 69 kb/s (default)
```

`-y` didn't help: the second run failed while parsing its options, before it
opened `out.mp4`, so the three-stream file from the first run is still there.

The failures don't all behave the same. Here is what the folder holds now:

```console
$ ls *.mp4
1.mp4
2.mp4
3.mp4
4.mp4
6.mp4
7.mp4
8.mp4
in.mp4
noaudio.mp4
out.mp4
$ stat -f '%N %z bytes' 7.mp4
7.mp4 0 bytes
```

The option error (`5.mp4`) created no file. The filtergraph error (`7.mp4`)
left an empty one. Check the exit code before you look at the file, or
delete the output first.

## Checklist

- Extra video stream after `-filter_complex`: look for `-map 0` (or `-map 1`
  …). It maps every stream of that input, including the video your filter
  already used.
- Keep the audio with `-map "[v]" -map 0:a`; use `-map "0:a?"` (quoted in
  zsh) if an input may have no audio.
- `-map 0 -map "[v]"` puts the unfiltered video first, with the `(default)`
  flag.
- `-map -0:v` after `-map "[v]"` gave `Filter ... has an unconnected output`.
  Put it before, or don't use it.
- List the streams of the result (`ffmpeg -i out.mp4 2>&1 | grep 'Stream #'`);
  the exit code is 0 either way.
- A failed run may leave the previous `out.mp4` untouched, or a 0-byte file.
  Check `$?` first.
- Filter addresses like `0x60000336c0a0` differ between runs.

*Tested on macOS 26.5.2 (Apple silicon) with ffmpeg 7.1, the static arm64
build shipped in the imageio-ffmpeg Python package (version line above), from
zsh. Test clips are generated with lavfi; no other ffmpeg builds tested.*
