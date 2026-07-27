# Sample pack prompt

A reusable prompt for handing to an LLM agent so it builds a sampler-ready pack
with this tool. Fill in the genre and tempo, paste the whole thing, and let it run.

It encodes the method — and the mistakes. The drone-fifth trap and the
phrase-vs-one-shot distinction each cost a full round when this pack was first
built.

---

Build me a sampler-ready sample pack using the `yt-sampler` CLI tool from this
repo. If you are working from another directory, call it by absolute path
(e.g. `~/dev/yt-sampler/yt-sampler`). Output to `~/Desktop/<PACK-NAME>/`.

## What I want

GENRE/VIBE: <e.g. boom bap hip hop>
TEMPO: <e.g. ~90 BPM>

Six subfolders:

| folder   | what        | how to cut                      |
|----------|-------------|---------------------------------|
| drums    | 3 loops     | 4 bars, tempo-matched           |
| chords   | 3 loops     | 4 bars, SAME tempo as drums     |
| bass     | 3 one-shots | single sustained notes, 4–5s    |
| lead     | 6 options   | 2 each from 3 DIFFERENT videos  |
| vocals   | 5 chops     | ~2s each                        |
| earcandy | 6 textures  | ~3s each                        |

Loops must be tempo-matched to each other. One-shots don't need to be — I'll
pitch-match those myself.

IMPORTANT: No sample can be more than 20 seconds long.

## The tool

Single Python file, no install. Wraps yt-dlp + ffmpeg + aubio.

```
yt-sampler <url> [START] [SECONDS]

-r, --random [N]   N random non-overlapping clips (default 5)
-d, --seconds N    clip length
-b, --bars N       cut to exactly N bars of 4/4 so clips LOOP seamlessly;
                   length is derived from the detected tempo
-o, --outdir DIR   output directory
--no-bpm           drop the BPM prefix from filenames
--note             detect the root note and lead the filename with it
--seed N           reproducible random picks
--dry-run          show the plan, download nothing
```

Output is 44.1k/16-bit stereo WAV, peak-normalized to -1 dBFS, starts snapped to
the nearest transient, edges micro-faded. Files are named BPM-first and
zero-padded (`090bpm__title__01m23s-01m43s.wav`) so `ls` sorts by tempo.

**ALWAYS QUOTE THE URL.** In zsh an unquoted `?` is a glob and the command dies
before it runs; an unquoted `&` silently truncates the URL and you get clips from
the wrong video.

Patterns:

```bash
yt-sampler "URL" -r 3 -b 8 -o ~/Desktop/pack/drums    # 3 x 8-bar loops
yt-sampler "URL" 23 4 --note --no-bpm -o ~/Desktop/pack/bass  # -> A1__...wav
yt-sampler "URL" -r 3 -d 2 --no-bpm -o ~/Desktop/pack/vocals
```

Use `-b` ONLY for loops. For one-shots use `-d` with `--no-bpm` — bar-locking a
single note is meaningless and a tempo prefix on it is noise.

## Finding sources

Search from the command line:

```bash
yt-dlp --no-playlist --print "%(id)s | %(duration)s | %(title).70s" \
  "ytsearch6:isolated drum break 90 bpm"
```

Title keywords that find ISOLATED material — this matters more than anything:

- `"drumless"` / `"backing track for drums"` → everything except drums
- `"for drum practice"` → bass and keys, no drums
- `"drums only"` / `"drum loop"` → drums alone
- BPM in the title is usually accurate; verify it with the tool anyway
- Tuning-reference videos ("Bass Guitar Tuner - E A D G") hold each note for 10+
  seconds — the best source for single-note one-shots

Check duration is long enough. 3 non-overlapping 8-bar clips at 90 BPM needs
~90s minimum.

## Rules learned the hard way

**One-shots are not short loops.** A 2-bar cut from a bass PERFORMANCE is a
phrase with several notes in it — useless for playing chromatically. You need a
source that holds notes individually: tuning references, instrument sound-demos,
note-by-note walkthroughs.

**Never use drone videos for pitched one-shots.** A "Cello Drone A" sounds the
fifth alongside the root. It measures as a perfect single pitch and is unusable.

**Prefer monophonic instruments** for leads (flute, single-note synth) — they
structurally can't sound an interval against themselves.

**Random placement fails on short or segmented sources.** If a video is a
sequence of distinct notes, map the note boundaries first, then use explicit
start times:

```bash
aubiopitch -u midi file.wav   # prints "time midi_note" per line
```

Bucket by second, find where the pitch changes, and target the stable middle of
each segment. This is how you get three DIFFERENT root notes instead of three
takes of the same one.

**Generate extras.** You cannot listen to any of this. Give me options and let me
pick — that's why lead is 6 clips from 3 videos, not 3 from one.

## Verify before you report

You can't hear it, so check what you can:

```bash
ffprobe -v error -show_entries format=duration -of csv=p=0 f.wav
ffmpeg -hide_banner -i f.wav -af volumedetect -f null - 2>&1 | grep volume
```

- **Length**: bar-locked clips must match `bars * 4 * 60 / bpm` within a few ms.
- **Not silent**: `mean_volume` below about -40 dB means sparse or broken.
- **Tempo**: read the BPM the tool put in the filename. If a chord source comes
  back at 150 against 90 drums, that's a 5:3 ratio and awkward to beat-match —
  throw it out and find another source rather than handing it to me.
- **Root note** (pitched one-shots): always cut these with `--note`, which writes the
  detected note into the filename. A clip named `xxx__` means the pitch wasn't steady
  enough to name — that's usually a phrase rather than a one-shot, so find a better
  source or a better start time.

**NEVER trust a note stated in a video title.** A pack was once built from videos titled
`a1-55-hz`, `c2-tuning-pitch-65-41-hz` and `d2-tuning-pitch-73-42-hz`, all cut at the same
offset on the assumption each held one constant tone. They sequence instead, and every
sample came out a whole tone off. `--note` measures what is actually in the clip; the
title is a guess.

Report the root notes — I need them to map the samples.

**Known limit, state it rather than papering over it:** pitch stability proves a
clip holds ONE pitch, not that nothing is stacked on top of it. It cannot detect
a fifth. Don't claim a sample is clean on that basis alone.

**If a video returns HTTP 403**, YouTube is blocking that one. Substitute a
different source; don't retry.

At the end, list every folder and file, and tell me each pitched one-shot's root
note and anything you'd flag.
