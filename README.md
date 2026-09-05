# opx-drift

Media timeline drift analyser for broadcast engineers.

Binary-only release. No source published.

## Download

**[→ Latest release](https://github.com/karhu-tv/opx-drift/releases/latest)**

| Platform | Requirements |
|----------|-------------|
| Linux x86_64 | Ubuntu 24.04 LTS · `sudo apt install ffmpeg` — FFmpeg 6.x, `libavformat.so.60`. 22.04 ships FFmpeg 4.4 and 26.04 ships FFmpeg 8; neither is supported. Other distributions need FFmpeg 6.x from their own sources |
| Windows x86_64 | Windows 10 (tested Aug 2026) · FFmpeg DLLs bundled |

## What it answers

**Does this file's presentation timeline hold together?** Two questions, two modes.

```
opx-drift file <media> [--stream <n>] [--rate <num>/<den>] [--json]
opx-drift sync <media> [<media-b>] [--video-stream <n>] [--audio-stream <n>]
               [--rate <num>/<den>] [--json]
opx-drift --version | -V
opx-drift --help    | -h

Exit codes:
  0  No discontinuities
  1  Discontinuities detected (in sync, also an R37 breach inside the file)
  2  Error, too little data, or a VFR stream
```

`--version` and `--help` are answered from any argument position, before the
subcommand is read, so `opx-drift file clip.mp4 --version` reports the version
rather than failing on a misplaced flag. (v0.2.0 had no `--version` at all;
added in 0.2.1.)

`file` — PTS deviation against the stream's own declared rate. The frame period is
resolved as an exact rational and never floated, so the reported deviation is not
an artefact of the measurement. Steps of ±1 period are dropped or repeated frames.

`sync` — the audio-versus-video differential, with EBU R37 (+40 / −60 ms) quoted.
Each track is measured against **its own** declaration, so the common-mode term
subtracts out. Given two files, their video timelines are compared instead.

## Examples

```bash
# Lip sync, with EBU R37 bounds
opx-drift sync programme.mxf

# PTS drift of a recorded stream
opx-drift file capture.ts

# Analyse a 30 fps file as though it were 29.97
opx-drift file capture.mp4 --rate 30000/1001

# Scriptable
opx-drift sync clip.mp4 && echo "OK" || echo "DRIFT"
```

## Sample output

Captured 2026-09-06 with opx-drift 0.2.2 on a 10 s synthetic bars file (H.264 + stereo PCM),
quoted verbatim. The earlier sample here had no surviving capture and was withdrawn.

```
bars_h264_1080p25_stereo_10s.mp4
  video  slope +0.000 ppm vs its own declaration · 250 pts · 10.0 s · max dev 0.00 ms · floor ±39.1 µs · SELF-CONSISTENT
  audio  slope +0.000 ppm vs its own declaration · 470 pts · 10.0 s · max dev 0.00 ms · floor ±10.4 µs · SELF-CONSISTENT
  no significant differential · differential +0.000 ppm (+ = audio ahead) · guard 14.836 ppm (3× the 4.945 ppm slope floor)
```

## Read this before quoting a number

A **common-mode** slope — both timelines drifting together against their
declarations — is a coherent retime. It plays fast or slow but stays **in sync**,
and it is not a sync fault. The differential is the finding. Judging absolute
speed needs an external reference that a file does not carry.

The quantisation floor is printed with every report. A slope inside it is labelled
quantisation-limited — `clean (quantisation-limited) · slope +0.000 ppm` — so a figure
that small is never read as a measured zero.

---

[karhu.tv](https://karhu.tv) · otso@karhu.tv
