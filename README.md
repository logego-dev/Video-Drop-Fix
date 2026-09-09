# FixDrops

Selective dropped-frame repair based on video timestamps.

**[Русская версия](README.ru.md)**

FixDrops analyzes frame presentation timestamps (PTS), estimates the nominal frame rate, and maps existing images onto a constant-frame-rate timeline. It interpolates empty frame slots instead of interpolating the entire video.

The script targets the original video timing and copies audio directly from the source without re-encoding. It compensates for intermediate-file timestamp offsets and checks the first output video timestamp after muxing.

> **Status:** Experimental. Test short clips and verify synchronization before using the output for multicamera editing or other timing-critical work.

## Features

- Nominal FPS estimation using the median frame interval.
- Manual FPS override, including fractional rates.
- Warnings when average FPS differs significantly from the median-based estimate.
- Dropped-frame interval reports with `HH:MM:SS:FF` timecodes.
- Selective interpolation of empty output frame slots.
- Optional interpolation at frame-assignment collisions.
- Optional half-rate output without changing playback speed.
- OpenCV DIS optical flow.
- NVIDIA NVENC or CPU encoding.
- HEVC and H.264 output.
- Source-bitrate targeting or quality-based encoding.
- Audio stream copy without intentional stretching or trimming.
- Intermediate timestamp-offset correction.
- Output video-start verification.
- JSON analysis and processing reports.

## Important limitations

- **Interpolation runs on the CPU.** NVENC accelerates encoding only.
- **RIFE is not included.**
- Existing images are not normally interpolated, but all output frames undergo color conversion and re-encoding.
- Duplicate images with regular timestamps are not detected.
- Optical flow may produce ghosting or deformation, especially around occlusions, fast motion, and long gaps.
- Exact arbitrary video duration cannot always be represented by a whole number of CFR frames. The video end is rounded up to a frame boundary.
- Automatic validation checks the video start and, when available, the container's frame count. It does not fully validate every timestamp or audio sample.
- Camera timecode/data streams, chapters, and general source metadata are not preserved by this version.

## Requirements

- Python 3.10 or newer.
- FFmpeg and FFprobe available in `PATH`.
- Python dependencies from `requirements.txt`.
- An NVIDIA GPU and compatible driver for NVENC encoding, or `--encoder cpu`.

The script targets progressive, square-pixel, 8-bit SDR video. HDR, interlaced video, non-square pixels, and rotation metadata are rejected by the current implementation.

### Hardware usage

| Operation | Hardware |
|---|---|
| Timestamp analysis and decoding | CPU |
| DIS optical flow | CPU |
| Full-resolution warping and blending | CPU |
| Encoding with `--encoder nvenc` | NVIDIA NVENC |
| Encoding with `--encoder cpu` | CPU |

Temporary encoded video is stored next to the output file. Allow enough disk space for both the temporary video and the final output.

## Installation

Download or clone the repository, then install the Python dependencies:

```bash
python -m pip install -r requirements.txt
```

Install FFmpeg separately and verify that both commands are available:

```bash
ffmpeg -version
ffprobe -version
```

## Quick start

### Analyze without creating a video

```bash
python fix_drops.py input.mp4 fixed.mp4 --analyze-only
```

This still writes a JSON report.

### Repair with automatic FPS detection

```bash
python fix_drops.py input.mp4 fixed.mp4
```

### Repair a known 60 FPS source

```bash
python fix_drops.py input.mp4 fixed.mp4 --source-fps 60
```

### Also interpolate collision slots

```bash
python fix_drops.py input.mp4 fixed.mp4 --source-fps 60 --collision-mode interpolate
```

### Faster optical-flow settings

```bash
python fix_drops.py input.mp4 fixed.mp4 --source-fps 60 --flow-width 640 --flow-preset fast
```

Quote paths containing spaces:

```bash
python fix_drops.py "D:\Videos\source clip.mp4" "D:\Videos\fixed clip.mp4"
```

The output directory must already exist. Existing output video files are not overwritten.

## How it works

### 1. Analyze timestamps

The script decodes the video and calculates adjacent frame intervals:

```text
interval = next PTS - previous PTS
median FPS estimate = 1 / median interval
average FPS = (frame count - 1) / (last PTS - first PTS)
```

Average FPS is used for diagnostics, not directly as the output frame rate.

### 2. Select a nominal frame rate

Automatic detection selects the standard rate closest to the median-based estimate:

```text
24000/1001, 24, 25, 30000/1001, 30,
50, 60000/1001, 60, 120000/1001, 120
```

If the nearest rate differs from the estimate by more than 5%, the script requests a manual override.

Nearby integer and fractional standards can be ambiguous. Use `--source-fps` when the recording standard is known.

This version does not use the camera's nominal-FPS metadata to resolve that ambiguity.

### 3. Report suspicious intervals

An interval is considered a suspected dropped-frame gap when:

```text
interval > 1.5 × median interval
```

The missing-frame count is estimated from the interval's ratio to the median.

Large intervals are also reported according to `--long-gap-ms`.

These are timestamp-based estimates, not proof that the camera physically lost specific frames.

### 4. Build a CFR timeline

Output presentation times are calculated from the first source video PTS:

```text
target PTS = first source video PTS + output frame index / output FPS
```

Source images are assigned sequentially to time bins around those target times:

- An occupied slot normally uses the nearest source image.
- An empty slot is interpolated from surrounding source frames.
- A collision slot follows `--collision-mode`.

An empty CFR slot is not necessarily a physical camera drop. Clock drift relative to the selected frame grid can also create empty or crowded slots.

### 5. Interpolate selectively

DIS optical flow is computed on reduced-resolution images. The original-resolution images are then warped and blended for the requested timestamp:

```text
alpha = (target PTS - left PTS) / (right PTS - left PTS)
```

At a suspected scene cut, the script holds the left frame instead of blending two scenes.

### 6. Encode, correct timestamps, and mux

The script:

1. Encodes the generated video into a temporary file.
2. Reads the first decoded video PTS from that file.
3. Calculates the offset required to match the source video start.
4. Muxes the corrected video with audio copied from the original input.
5. Checks the first output video PTS.

This avoids assuming that an encoded intermediate file always starts at zero.

## Command-line options

```text
python fix_drops.py INPUT OUTPUT [options]
```

### Timeline and analysis

| Option | Default | Description |
|---|---|---|
| `--source-fps` | `auto` | Nominal FPS or automatic estimation |
| `--half-fps` | Off | Divide the nominal rate by two |
| `--collision-mode` | `nearest` | `nearest` or `interpolate` |
| `--warn-percent` | `5` | Warning threshold for average/median FPS disagreement |
| `--long-gap-ms` | `100` | Large-interval warning threshold |
| `--analyze-only` | Off | Analyze and write JSON without encoding |

The FPS disagreement percentage is:

```text
abs(average FPS - median FPS estimate) / median FPS estimate × 100
```

`--long-gap-ms` only controls warnings and reporting. It does not disable interpolation across long gaps.

### Interpolation

| Option | Default | Description |
|---|---|---|
| `--flow-width` | `960` | Width used for optical-flow estimation |
| `--flow-preset` | `medium` | `fast` or `medium` |
| `--scene-threshold` | `0.65` | Histogram-based scene-cut threshold, from 0 to 1 |

Reducing `--flow-width` does not reduce output resolution.

A lower scene threshold causes more pairs to be treated as scene cuts. The detector is approximate and can both miss cuts and reject valid motion.

### Encoding

| Option | Default | Description |
|---|---|---|
| `--encoder` | `nvenc` | `nvenc` or `cpu` |
| `--codec` | `hevc` | `hevc` or `h264` |
| `--rate-control` | `source` | `source` bitrate target or `quality` mode |
| `--bitrate-mbps` | Unset | Explicit target video bitrate |
| `--cq` | `19` | NVENC CQ or CPU CRF in quality mode |
| `--nvenc-preset` | `p4` | NVENC preset, `p1` through `p7` |

`--bitrate-mbps` overrides `--rate-control`.

In source-bitrate mode, the script reads the video stream bitrate from metadata. If unavailable, specify a bitrate manually or use quality mode.

The bitrate is a VBR target, not a guarantee of an identical file size or measured average bitrate.

`--cq` is ignored when a bitrate target is used. Lower CQ/CRF values generally increase quality and file size, but values are not directly comparable across encoders.

## Collision modes

### `nearest`

Selects the source image closest to the output timestamp.

This minimizes synthesis but can leave a small local motion irregularity when source images must be discarded.

### `interpolate`

Synthesizes an image at the target timestamp for collision slots.

It replaces the image in an existing slot; it does not add another output frame or extend the timeline.

With `--half-fps`, two source frames in a bin are treated as normal downsampling. A collision is reported when there are more than two.

## Examples

### Fractional 59.94 FPS

```bash
python fix_drops.py input.mp4 fixed.mp4 --source-fps 60000/1001
```

### 60 to 30 FPS without changing playback speed

```bash
python fix_drops.py input.mp4 fixed.mp4 --source-fps 60 --half-fps
```

### Explicit 76 Mbps target

```bash
python fix_drops.py input.mp4 fixed.mp4 --bitrate-mbps 76
```

### Quality-based encoding

```bash
python fix_drops.py input.mp4 fixed.mp4 --rate-control quality --cq 18
```

### H.264 output

```bash
python fix_drops.py input.mp4 fixed.mp4 --codec h264
```

### CPU encoding

```bash
python fix_drops.py input.mp4 fixed.mp4 --encoder cpu --rate-control quality --cq 18
```

### More detailed optical flow

```bash
python fix_drops.py input.mp4 fixed.mp4 --flow-width 1280 --flow-preset medium --collision-mode interpolate
```

## Timing and duration

The script does not deliberately retime the video by assigning consecutive restored frame numbers while ignoring source PTS. Output positions remain anchored to the source timeline.

Individual source images may nevertheless be shown slightly earlier or later because of CFR quantization.

The video duration is rounded up:

```text
output frame count = ceil(source video duration × output FPS)
output duration = output frame count / output FPS
```

For example:

```text
Source video duration: 4.410389 s
Output rate:          60 FPS
Output frame count:   265
Output duration:      4.416667 s
```

Audio is not trimmed to match this rounded video end.

Complex container edit lists and application-specific playback behavior are not exhaustively handled. The script also does not correct clock drift already present between independently recorded cameras.

### Output validation

After muxing, the first output video PTS is compared with the source start.

The tolerance is the greater of:

- 1.1 milliseconds;
- two units of the output video time base.

If the container reports a frame count, it is also checked.

A failed check sets:

```json
"timing_check": "FAILED"
```

The output file may still exist after failure. Do not treat it as validated.

## Reports and timecodes

For `fixed.mp4`, the report is:

```text
fixed.mp4.drops.json
```

It contains:

- Input and output FPS information.
- Median and average timing statistics.
- Warnings and suspicious intervals.
- Estimated missing-frame counts.
- Planned output duration and frame count.
- Intermediate timestamp compensation.
- Output-start verification results.
- Processing counters.

Processing counters include:

| Field | Meaning |
|---|---|
| `original` | Output frames using a selected source image |
| `interpolated` | Synthesized frames |
| `edge_hold` | Holds where a second reference is unavailable |
| `scene_hold` | Holds at suspected scene cuts |
| `empty_slots` | Empty output-grid bins |
| `collision_slots` | Crowded bins requiring collision handling |

Reports use relative **non-drop-frame (NDF)** timecodes:

```text
HH:MM:SS:FF
```

Timecodes start at the first source video frame and use the nominal source rate, even with `--half-fps`.

They are not embedded camera timecodes. At fractional rates, NDF timecode does not remain identical to wall-clock time.

The report file may be overwritten by another run using the same output name.

## Audio verification

Audio is copied without re-encoding, but this version does not automatically compare decoded audio samples.

To compare the first audio stream manually:

```bash
ffmpeg -v error -i input.mp4 -map 0:a:0 -vn -c:a pcm_s32le -f hash -hash sha256 -
```

```bash
ffmpeg -v error -i fixed.mp4 -map 0:a:0 -vn -c:a pcm_s32le -f hash -hash sha256 -
```

Matching hashes indicate matching decoded sample sequences in this comparison.

**Audio hashes do not verify audio placement relative to video.** Check timestamps and playback synchronization separately.

## DaVinci Resolve

- Import the result as a new clip, preferably with a new filename.
- Verify that Resolve detects the expected FPS.
- Use a matching timeline rate for frame-by-frame validation.
- Check the beginning, end, and several repaired gaps.
- Do not reinterpret a 60 FPS clip as 24 or 30 FPS in Clip Attributes unless a playback-speed change is intended.
- Different waveform rendering alone does not prove different audio samples. Check clip placement, processing, and waveform caching.

## Troubleshooting

### Input file not found

Use the actual filename or an absolute path. The script does not search other directories automatically.

### Output file already exists

Choose another output name or remove the previous result.

### NVENC failure

Check the NVIDIA driver and FFmpeg build, or try:

```text
--encoder cpu
```

### Source bitrate is missing

Use:

```text
--bitrate-mbps 50
```

or:

```text
--rate-control quality --cq 19
```

### Audio codec is incompatible with MP4

Try an MKV output. Verify compatibility with your editor separately.

### Processing is slow

Try:

```text
--flow-width 640 --flow-preset fast
```

Optical flow, image warping, decoding, and color conversion still use the CPU. Smaller flow images do not proportionally accelerate the entire pipeline.

### Timing validation failed

Keep the console output and JSON report. Do not use the result for timing-critical work until the discrepancy is understood.

## Reporting issues

Include:

- The command used.
- OS, Python, FFmpeg, and dependency versions.
- Input codec, resolution, and expected FPS.
- Console output and JSON report.
- A short reproducible sample, if you have permission to share it.

Review reports before publishing them: they contain file paths and filenames.

## License

The project code is licensed under the [MIT License](LICENSE).

FFmpeg, PyAV, OpenCV, NumPy, and other dependencies are distributed under their respective licenses. This project's license does not replace their license terms.