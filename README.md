# opx-drift

Media timeline drift analyser for broadcast engineers.

Binary-only release. No source published.

## Download

**[→ Latest release](https://github.com/karhu-tv/opx-drift/releases/latest)**

| Platform | Requirements |
|----------|-------------|
| Linux x86_64 | Ubuntu 24.04 LTS · `sudo apt install ffmpeg` — FFmpeg 6.x, `libavformat.so.60`. 22.04 ships FFmpeg 4.4 and 26.04 ships FFmpeg 8; neither is supported. Other distributions need FFmpeg 6.x from their own sources |

**Windows x86_64: withdrawn.** The Windows zips bundled FFmpeg 6.1.1 DLLs (gyan.dev full_build-shared), licensed GPL version 3 or later. The zips did not include that licence's text or an offer of the FFmpeg source, which the licence requires. Windows downloads withdrawn 29 Sep 2026 for this reason. The Linux build does not bundle FFmpeg.

## Known issues in 0.2.2

*Known issues added 29 Sep 2026.* These items are also listed on the [latest release](https://github.com/karhu-tv/opx-drift/releases/latest) and on [karhu.tv](https://karhu.tv/tools.html#opx-drift-issues).

- **`sync` measures drift, not a constant offset.** Drift means a change in A/V alignment across the file. Each track is anchored at its own first packet, so a file whose audio is late by a fixed amount from the first frame reports no differential and exits 0. On 29 Sep 2026 a copy of our bars file with its audio 178 ms late passed. `opx-probe --house` flags such a file as a non-zero stream start (reported as an edit list).
- **Dropped frames mid-file.** When dropped or repeated frames sit near the middle of a file, or are spread through it, the slope estimate absorbs them. The report then shows a slope the file does not have (roughly drops ÷ frames) and, on MPEG-TS and on some MP4s (25 fps in our tests), lists nearly every frame as a sub-period step. The ±1-period steps are still listed and the exit code is still 1. In `sync` the same slope appears as an A/V differential and can be reported as an EBU R37 crossing. On 29 Sep 2026 one dropped frame at the midpoint of a 500-frame 50p TS read +2066 ppm and 499 steps.
- **AAC in MKV or FLV.** `sync` reports a false audio drift of about 1.6 %, and an EBU R37 breach, on AAC audio in Matroska (MKV) or FLV, because those containers round each AAC packet's duration to the millisecond. A lossless remux of a clean MP4 failed this way on 29 Sep 2026. In the same tests AAC in MP4, MOV and TS read clean, as did PCM audio in MXF and Opus audio in WebM.
- **VFR.** VFR is recognised only when FFmpeg reports no frame rate. A variable-rate file that declares a nominal rate is analysed at that rate: `file` reports steps and exits 1, not 2, and `sync` can report a false R37 breach.
- **Truncated files.** A truncated file is analysed as far as it goes and can read clean; the report does not say the file ended early.
- **`--audio-stream`.** An index that is out of range or not audio falls back to video only, reports "no audio stream in this file", and exits 0.
- **Exit code 101.** opx-drift ends in a Rust panic with exit code 101, outside the documented 0/1/2, when a file name is not valid UTF-8, and when stdout is closed before the report is written (for example `| head` on a long report).
- **Publisher name.** `--help` names a limited company that is not the publisher, and the archive's README.txt runs the trading name together as one word. The publisher is Bear Media Services LTD, trading as Karhu TV. Both are fixed in source; the published archives are unchanged.
- **Supported systems.** The README.txt inside the Linux archive says `sudo apt install ffmpeg (Ubuntu/Debian)`. No current Debian release ships FFmpeg 6; of Ubuntu releases, only 24.04 LTS is supported.

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
resolved as an exact rational and never floated, so rounding the period adds no
error. Steps of ±1 period are dropped or repeated frames.

`sync` — the audio-versus-video differential, with EBU R37 (+40 / −60 ms) quoted.
Each track is measured against **its own** declaration, so the common-mode term
subtracts out. Given two files, their video timelines are compared instead.

## Examples

```bash
# Lip-sync DRIFT (not a constant offset), with EBU R37 bounds
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
