# How Computers See and Hear: Video, Audio, and FFmpeg Explained Simply

What is a video file, actually? And how does a computer take what you see and hear and turn it into something it can store, move, and play back?

## A video is just a flipbook

Remember flipbooks — a stack of paper, a slightly different drawing on each page, and when you flip through fast enough, it looks like the drawing is moving?

That's exactly what video is. A video file is a big stack of individual photos, shown one after another, fast enough that your brain stitches them into motion. Each photo is called a **frame**.

**FPS (frames per second)** is just: how many pages does this flipbook flip per second?

- 24 fps — the traditional film standard. Flip 24 photos every second.
- 30 fps — common for everyday video, feels a bit smoother.
- 60 fps — very smooth, common in games and sports footage.

Fewer photos per second = choppier motion (each photo has to "hold" longer). More photos per second = smoother motion, but more photos to store.

## Resolution is just how detailed each photo is

Zoom into any digital photo far enough and you'll see it's made of tiny colored squares — **pixels**. Resolution is just: how many of those tiny squares make up the picture?

"1920×1080" means 1,920 squares across, 1,080 squares down. More squares packed into the same picture = finer detail = sharper image. Same idea as a mosaic — a mosaic made of thousands of tiny tiles can show a detailed face; a mosaic made of ten big tiles can only show a rough blob.

## Sound is a wiggle, and computers store the wiggle as numbers

Sound is air vibrating — pushing and pulling in a wave. A microphone is just a tiny membrane that wiggles along with that wave and turns the wiggle into an electrical signal.

A computer can't store a continuous wiggle directly — computers only understand discrete numbers. So it does something clever: it measures the height of the wiggle thousands of times per second, and writes down each measurement as a number.

- **Sample rate** — how many measurements per second. CD-quality audio measures the wiggle 44,100 times every second (44.1kHz). More measurements = a more faithful copy of the original wiggle.
- **Bit depth** — how precise each individual measurement is. 16-bit audio can describe 65,536 different heights for the wiggle at each measurement; more bits, more precision.

Audio is genuinely just: a very long list of numbers, each one a snapshot of how far the wiggle had traveled at that instant.

## How transcription actually works (the model itself)

Remember from just above: sound is a wiggle, and a computer stores it as a long list of numbers — thousands of tiny measurements per second.

A speech recognition model like Whisper is something that was shown an enormous pile of these number-lists paired with the correct written-down words — millions of hours of people talking, with transcripts already attached. During that training, it slowly learned the patterns: this specific shape of wiggle, repeated this way, tends to mean the word "hello." It never memorized your specific audio — it learned the general shape of how spoken words turn into sound waves, across thousands of voices, accents, and speaking speeds.

So when you hand it a new audio file it's never heard, it isn't looking anything up — it's doing the same kind of prediction it learned to do: sliding through the audio in small windows, and for each window asking "given everything I learned about how speech sounds, what words is this most likely to be?" It keeps doing that, window after window, stitching the predictions into a full transcript. That's genuinely the whole trick — it's pattern-matching at a very large scale, not understanding language the way a person does.

**Word-level timestamps** are just the model also keeping track of exactly where in the audio each predicted word started and stopped — like it's running a stopwatch the entire time, and jotting down the reading every time it decides "a new word just started here."

**Segments** are the model's own grouping of consecutive words into a sentence-or-phrase-sized chunk, usually based on pauses or natural breaks in speech — it's not you cutting the transcript up afterward, the model hands it back pre-grouped.

## Why raw video and audio are enormous

Now put it together. One second of 1080p video at 30fps is 30 full photos, each made of over 2 million pixels, each pixel needing a few numbers to describe its color. One second of CD-quality stereo audio is 44,100 × 2 numbers (two channels). Stored with zero cleverness, a couple of minutes of video would be gigabytes.

This is the actual problem codecs exist to solve.

## Codecs: a shared shortcut language

A **codec** (**co**der/**dec**oder) is an agreed-upon set of tricks two computers use to make something much smaller to store, and then rebuild it later.

Think about describing a coloring book page to a friend over the phone. You *could* read out the color of every single pixel, one by one. Or you could say "the whole top half is blue sky, the bottom half is green grass, there's a red circle in the middle" — a shortcut description that reconstructs almost the same picture using a tiny fraction of the words. That second approach is basically what a codec does — it notices patterns and redundancy (a mostly-still background across many frames, large blocks of a single color, repeating textures) and describes those patterns compactly instead of listing every raw pixel or every raw sound sample.

Two flavors worth knowing:
- **Lossy** — throws away detail a human is unlikely to notice, to save a lot more space. MP3 (audio) and H.264 (video) are lossy.
- **Lossless** — shrinks the file with clever tricks but keeps every original detail exactly. FLAC (audio) and PNG (images) are lossless.

## Containers: the box the codec's output rides in

A **container** (`.mp4`, `.mov`, `.wav`, `.mkv`) is just a labeled box. It holds one or more compressed tracks — a video track, an audio track, maybe subtitles — plus a table of contents describing what's inside and how the pieces line up in time. The container itself doesn't compress anything; it's the packaging.

The **codec** is what actually did the compressing *inside* that box.

So a `.mp4` file is a box that commonly holds H.264-compressed video and AAC-compressed audio. A `.wav` file is a box that conventionally holds *uncompressed* audio (PCM) — no audio codec doing any compression trick at all, just the raw number list, wrapped in a `.wav`-shaped label.

## Where FFmpeg fits into all of this

FFmpeg is a universal translator for containers and codecs. It knows how to open almost any box, understand almost any codec's compression trick, and either:
- **Inspect** — read the label and table of contents without touching what's inside (that's `ffprobe`).
- **Transcode** — decompress using one codec's rules, then recompress using a different codec's rules, and repackage into a new box .
- **Remux** — just repackage the same compressed data into a different box, with no decompress/recompress step at all (much faster, since no quality-affecting re-encoding happens)

## Practical FFmpeg: from concepts to commands

Everything above was building toward this: FFmpeg's command-line shape is always the same skeleton —

```bash
ffmpeg [input options] -i input_file [output options] output_file
```

Read left to right: how to open the input box, which box to open, what to do before closing it into the output box, which box to close it into.

**Inspecting a file (what `ffmpeg.probe()` runs under the hood):**
```bash
ffprobe -v quiet -print_format json -show_format -show_streams input.mp4
```
That's the literal command `ffmpeg-python`'s `.probe()` shells out to and parses for you. Worth running it directly in a terminal once against a real file — seeing the raw JSON it returns makes it obvious where `format_info` and `streams` in your Python code are actually coming from.

**Trimming a clip — two ways, with a real tradeoff between them:**
```bash
# Fast: no re-encoding, just repackaging (remux)
ffmpeg -ss 00:01:30 -t 00:00:15 -i input.mp4 -c copy output.mp4

# Frame-accurate: re-encodes, slower, but cuts exactly where asked
ffmpeg -i input.mp4 -ss 00:01:30 -t 00:00:15 -c:v libx264 -c:a aac output.mp4
```
The difference is `-ss` placement and `-c copy`. `-c copy` means "don't decompress and recompress, just copy the compressed bytes as-is" — very fast, but it can only cut at a **keyframe** (a frame stored as a complete image rather than "the changes since the last frame" — most frames in a compressed video are the latter, and can't be cut at cleanly on their own). Put `-ss` after `-i` with real re-encoding, and FFmpeg decodes from the true start and can cut at the exact frame you asked for — slower, because it's now doing the full decompress/recompress cycle.

**Extracting audio :**
```bash
ffmpeg -i input.mp4 -vn -acodec pcm_s16le output.wav
```
`-vn` = "no video" — audio-only output. `pcm_s16le` = uncompressed audio, matching what a `.wav` container conventionally expects — exactly the codec/container pairing from the earlier bug fix.

**Grabbing a single frame as a thumbnail:**
```bash
ffmpeg -i input.mp4 -ss 00:00:05 -vframes 1 thumbnail.jpg
```
`-vframes 1` = stop after exactly one frame.

**Resizing (relevant later, for auto-reframing to vertical/short-form):**
```bash
ffmpeg -i input.mp4 -vf scale=1080:1920 output.mp4
```
`-vf` = video filter; `scale=width:height` is one of dozens of available filters.

**Other common tasks people reach for FFmpeg to do — quick reference:**

| Task | Command | What's happening |
|---|---|---|
| Convert between formats | `ffmpeg -i input.mov output.mp4` | Just naming a different output extension tells FFmpeg to remux (and re-encode if needed) into that container |
| Compress a video | `ffmpeg -i input.mp4 -c:v libx264 -crf 28 output.mp4` | `-crf` (Constant Rate Factor) trades quality for size on a roughly 0–51 scale — 18 is close to lossless, 23 is a common default, 28+ is visibly more compressed |
| Merge multiple clips | `ffmpeg -f concat -safe 0 -i filelist.txt -c copy output.mp4` | `filelist.txt` lists each clip on its own line (`file 'clip1.mp4'`); fastest way to join clips that already share the same codec/resolution |
| Normalize frame rate | `ffmpeg -i input.mp4 -r 30 output.mp4` | Useful before merging clips recorded at mismatched frame rates |
| Make a shareable GIF | `ffmpeg -i input.mp4 -ss 5 -t 3 -vf "fps=10,scale=480:-1" output.gif` | Grabs a 3-second window, drops the frame rate and size down — full-fps, full-res GIFs are enormous |
| Extract one frame per second | `ffmpeg -i input.mp4 -vf fps=1 frame_%04d.png` | Handy for building a preview contact-sheet or montage from a long video |
| Mute a video | `ffmpeg -i input.mp4 -an output.mp4` | `-an` = drop the audio stream entirely, video untouched |
| Add a watermark/logo | `ffmpeg -i input.mp4 -i logo.png -filter_complex overlay=10:10 output.mp4` | Two inputs; the filter positions the second image over the first at pixel coordinates (10, 10) |
| Adjust volume | `ffmpeg -i input.mp4 -filter:a "volume=1.5" output.mp4` | `1.5` = 150% of original loudness; `0.5` would halve it |
| Batch-convert a whole folder | `for f in *.mov; do ffmpeg -i "$f" "${f%.mov}.mp4"; done` | A shell loop, not an FFmpeg feature itself — but this is how most people actually process more than one file at a time |

Two you'll see mentioned often but that get genuinely fiddly fast: **changing playback speed** (needs `setpts` for video and `atempo` for audio, kept in sync separately) and **burning in subtitles** (needs a filter pointing at a subtitle file, and font rendering can behave differently per system). Both are real, well-documented FFmpeg capabilities — just not one-liners worth memorizing here; look them up when you actually need them.

**Reading `ffmpeg-python`'s fluent chain as the CLI command it generates:**
```python
(
    ffmpeg
    .input('input.mp4', ss=90, t=15)
    .output('output.mp4', c='copy')
    .run()
)
```
This builds and runs roughly:
```bash
ffmpeg -ss 90 -t 15 -i input.mp4 -c copy output.mp4
```
Whenever the Python chain does something confusing, translate it to the CLI form in your head first — the library is a thin wrapper generating exactly this command line, nothing more magical underneath.

## The core mental model: translating any CLI command into `ffmpeg-python`

Once the CLI skeleton is second nature — `ffmpeg [input options] -i input [output options] output` — translating it into `ffmpeg-python` stops being guesswork and becomes mechanical. There are really only three buckets:

| CLI position | `ffmpeg-python` call | What goes here |
|---|---|---|
| flags **before** `-i` | `ffmpeg.input(path, **kwargs)` | seeking (`ss`), format hints — anything about *reading* the input |
| flags **after** the input (up to the output filename) | `ffmpeg.output(path, **kwargs)` | codecs, duration, filters, bitrate — anything about *writing* the output |
| a verb like `-c copy`, `-vn`, `-an` | a kwarg inside `.output()` | `c='copy'`, `vn=None`, `an=None` |

**The translation algorithm:**

1. Scan the CLI command left to right.
2. Everything between `ffmpeg` and `-i` → becomes kwargs in `.input()`.
3. Everything between the input file and the output file → becomes kwargs in `.output()`.
4. Flag names lose their leading dash: `-ss` → `ss=`, `-t` → `t=`, `-c` → `c=`. Flags with a colon (`-c:v`, `-c:a`) can't be written as a normal kwarg, so they go in a dict: `**{'c:v': 'libx264', 'c:a': 'aac'}`.
5. Valueless/boolean flags (`-vn`, `-an`, `-y`) get `None` as their value: `vn=None`.
6. Chain `.input()` → `.output()` → `.run()`.

**Worked example**, using the fast trim command from above:

```
ffmpeg -ss 00:01:30 -t 00:00:15 -i input.mp4 -c copy output.mp4
```

Walking it left to right: `-ss` and (in this ordering) `-t` sit before `-i`, so they're input kwargs; `-c copy` sits after, so it's an output kwarg.

```python
import ffmpeg

(
    ffmpeg
    .input('input.mp4', ss='00:01:30', t='00:00:15')
    .output('output.mp4', c='copy')
    .run()
)
```

Note `ss` and `t` accept `'HH:MM:SS'` strings directly, just like on the CLI — no need to convert timestamps to raw seconds if the clock-time format is easier to reason about.

**Why the bucket placement isn't just cosmetic.** `-ss` is the one flag where *which bucket it lands in* changes behavior, not just syntax — this is the same remux-vs-transcode tradeoff from the trimming example earlier:

- `-ss` **before** `-i` → `ffmpeg.input(path, ss=...)` → fast seek: jumps straight to the nearest keyframe via the container's index, no decoding needed to get there.
- `-ss` **after** `-i` → `ffmpeg.output(path, ss=...)` → accurate seek: decodes from the true start of the file and discards frames until it reaches your timestamp — slower, but frame-exact.

So the frame-accurate trim command from earlier —

```bash
ffmpeg -i input.mp4 -ss 00:01:30 -t 00:00:15 -c:v libx264 -c:a aac output.mp4
```

— translates with `ss` moved into `.output()`, since it now comes after `-i` on the CLI:

```python
(
    ffmpeg
    .input('input.mp4')
    .output('output.mp4', ss='00:01:30', t='00:00:15', **{'c:v': 'libx264', 'c:a': 'aac'})
    .run()
)
```

Same flag name, different bucket, different underlying behavior — the position in the CLI command is the signal, not the flag itself.

**A built-in sanity check.** `ffmpeg-python` can print the exact CLI command it's about to run, which makes it easy to confirm a translation is correct — just compile and compare token-by-token against the original:

```python
node = ffmpeg.input('input.mp4', ss='00:01:30').output('output.mp4', t='00:00:15', c='copy')
print(node.compile())
# ['ffmpeg', '-ss', '00:01:30', '-i', 'input.mp4', '-t', '00:00:15', '-c', 'copy', 'output.mp4']
```

**Quick reference for flags that show up constantly:**

| CLI flag | Bucket | Python kwarg |
|---|---|---|
| `-ss` (fast seek) | input | `ffmpeg.input(path, ss=...)` |
| `-ss` (accurate seek) | output | `.output(path, ss=...)` |
| `-t` (duration) | output | `t=...` |
| `-to` (end timestamp) | output | `to=...` |
| `-c copy` / `-c:v libx264` | output | `c='copy'` / `**{'c:v': 'libx264'}` |
| `-vn` / `-an` | output | `vn=None` / `an=None` |
| `-r 30` (fps) | output | `r=30` |
| `-vf "scale=..."` | output | `vf='scale=...'` |
| `-crf 23` | output | `crf=23` |

Once "before-`-i`-vs-after" clicks as the organizing question, almost any command from the reference table above — or from the [ffmpeg-python examples folder](https://github.com/kkroening/ffmpeg-python/tree/master/examples) — becomes something to translate on sight rather than something to look up.

## Quick glossary

| Term | Plain meaning |
|---|---|
| Frame | One still photo in the video flipbook |
| FPS | How many photos flip past per second |
| Resolution | How many tiny colored squares make up each photo |
| Sample rate | How many times per second the sound wiggle gets measured |
| Bit depth | How precisely each sound measurement is stored |
| Codec | The shortcut language used to shrink (and later rebuild) the video/audio data |
| Container | The labeled box holding the compressed tracks together (the file format itself) |
| Bitrate | Roughly: how many of those shortcut-language "words" are used per second — higher usually means better quality, but bigger files |
| Keyframe | A frame stored as a complete image; other frames just store "what changed since the last one" and can't be cut at cleanly on their own |
| Transcode | Decompress with one codec, recompress with another |
| Remux | Repackage into a different box without touching the compression at all |

Once containers and codecs stop being tangled together in your head, an FFmpeg command stops looking like a spell — it's just: "open this box, do this specific thing to what's inside, close it into that box."
