# yt-sampler

Turn a YouTube link into sampler-ready WAV clips.

Give it a URL and it writes the whole video into `./samples/` as a WAV. Add `-r` and it
scatters random clips through the video instead. Either way the BPM leads the filename,
so a plain `ls` sorts your samples by tempo:

```
134bpm__children-robert-miles-handpan-cover__00m04s-00m24s.wav
139bpm__children-robert-miles-handpan-cover__00m48s-01m08s.wav
140bpm__children-robert-miles-handpan-cover__01m35s-01m55s.wav
144bpm__children-robert-miles-handpan-cover__03m52s-04m12s.wav
```

It's a wrapper — `yt-dlp` fetches, `ffmpeg` slices and normalizes, `aubio` reads tempo.
The script is a single stdlib-only Python file with no packages to install.

## Install

Needs Python 3.10+ and three command-line tools:

```bash
brew install yt-dlp ffmpeg aubio
```

On Debian/Ubuntu the equivalents are `yt-dlp`, `ffmpeg` and `aubio-tools`.

`aubio` is optional — without it clips are still produced, just without BPM in the name.
`yt-dlp` and `ffmpeg` are required, and the script exits with a clear message if either
is missing.

Then grab the script and make it executable:

```bash
git clone https://github.com/YOUR-USERNAME/yt-sampler.git
cd yt-sampler
chmod +x yt-sampler
./yt-sampler --help
```

To run it from anywhere, symlink it onto your `PATH`:

```bash
ln -s "$PWD/yt-sampler" ~/.local/bin/yt-sampler
```

## Usage

```bash
yt-sampler "https://youtu.be/VIDEO_ID"              # the whole video
yt-sampler "https://youtu.be/VIDEO_ID" 1:23         # one 20s clip at 1:23
yt-sampler "https://youtu.be/VIDEO_ID" 1:23 8       # one 8s clip at 1:23
yt-sampler "https://youtu.be/VIDEO_ID" -r           # 5 random 20s clips
yt-sampler "https://youtu.be/VIDEO_ID" -r 10 -d 15  # 10 random 15s clips
```

Times accept `90`, `1:30`, `1:02:03` or `1m30s`.

**Always quote the URL.** YouTube links contain `?` and `&`, both of which the shell
treats specially. In zsh an unquoted `?` is a glob pattern and the command fails before
it even runs:

```
zsh: no matches found: https://www.youtube.com/watch?v=oMRjbkHVrxk
```

An unquoted `&` is worse — it doesn't error, it silently truncates. A playlist or
timestamp link like `...watch?v=abc&list=PLxyz` arrives as just `...watch?v=abc`, and
you get clips from the wrong video with no warning at all. Quoting prevents both.

Running `yt-sampler` with no arguments prints the usage block and these examples.

### Options

| Flag | Default | |
|---|---|---|
| `-r, --random [N]` | off | take N random non-overlapping clips instead of the whole video (N defaults to 5) |
| `-d, --seconds` | `20` | clip length |
| `-s, --start` | | start time, flag form |
| `-o, --outdir` | `samples` | output directory |
| `--sr` | `44100` | sample rate |
| `--bits` | `16` | bit depth, `16` or `24` |
| `--mono` | off | downmix to mono |
| `--peak` | `-1.0` | target peak in dBFS |
| `--no-normalize` | | skip peak normalization |
| `--no-bpm` | | skip BPM detection |
| `--seed` | | seed the RNG so random picks repeat |
| `--cookies-from-browser` | | pass browser cookies to yt-dlp |
| `--dry-run` | | show what would be made, download nothing |

### Bot-checked or age-gated videos

YouTube sometimes refuses anonymous downloads. Point yt-dlp at your browser's cookies:

```bash
yt-sampler "https://youtu.be/VIDEO_ID" --cookies-from-browser chrome
```

Or set it once: `export YTS_COOKIES_BROWSER=chrome`

## Notes

**Filenames lead with BPM**, zero-padded to three digits so `086bpm` sorts before
`130bpm` — without the padding a plain string sort puts 130 first. When the tempo can't
be determined the file is named `xxxbpm__…`, which sorts after everything numeric. With
`--no-bpm` the prefix is dropped entirely and names start with the title.

**Random clips never overlap.** Ask for 5 and you get 5 distinct sections, not two
near-duplicates a second apart. If the video is too short to fit that many
non-overlapping clips, you get as many as do fit, with a warning — a 99-second video
holds four 20-second clips, not five.

**Peak normalization** is on by default and scales each clip so its loudest peak sits at
−1 dBFS. It's peak-only — it changes gain, never dynamics, so transients survive intact.
Turn it off with `--no-normalize`.

**BPM** comes from `aubiotrack` beat timestamps: outlier gaps (a dropped or doubled beat)
are discarded, and the tempo is the average of what remains. If too few beats are found,
or too few gaps agree with each other, the reading is discarded and `aubio tempo` is tried
as a fallback; if that also fails the BPM is simply left off the filename. Measured against
synthesized known-tempo material it lands within about 1 BPM. On sparse or ambient passages
expect it to report half or double time — it folds results into a 70–180 range, so a 60 BPM
passage will read as 120.

**The source audio is not cached.** Each run re-downloads, so re-rolling random clips on a
long video costs another download.

## What you're downloading

Pulling media off YouTube is against YouTube's Terms of Service, and the audio you get is
almost always someone else's copyrighted work. Whether any particular sample is fair use
depends on how much you take, what you do with it, and where you live — and clearing a
sample before you release anything commercially is on you. This tool doesn't check any of
that. It just moves audio around.

The `samples/` directory is in `.gitignore` for the same reason: don't commit other
people's music to a public repo.

## Possible next steps

- Beat-aligned clip lengths — cut to exactly 4/8/16 bars so clips loop seamlessly
- Onset-snapped starts, so a clip begins on a transient instead of mid-note
- Short fades to remove boundary clicks
- Caching downloads by video ID
- Silence-skipping when picking random windows
- Key detection; stem separation via `demucs`

## License

MIT — see [LICENSE](LICENSE).
