# Video Translator (Whisper + VAD Retiming)

[Open in Colab](https://colab.research.google.com/drive/1jgSBhdLgjRd--2-HNqMwcAbfjee5fBWy#scrollTo=aQSVKuuaTxtL)

A Google Colab notebook that takes a video or audio file, transcribes and translates its speech into English subtitles, and then re-times those subtitles to match actual speech activity in the audio (instead of Whisper's raw segment timestamps).

## What it does

1. **Environment setup** — Installs `faster-whisper` and `ffmpeg-python`, installs the `ffmpeg` binary, and checks for GPU (CUDA) availability.
2. **File upload & audio extraction** — Uses the Colab file upload widget to accept a file. If it's a video (`.mp4`, `.mov`, `.mkv`, `.avi`, `.wmv`, `.flv`, `.webm`), it extracts the audio track with `ffmpeg`. If it's already an audio file (`.mp3`, `.wav`, `.m4a`, `.aac`, `.flac`, `.ogg`, `.opus`), it's used as-is.
3. **Model loading** — Loads OpenAI Whisper's `large-v3` model via `faster-whisper`, using GPU (`float16`) if available, otherwise CPU (`int8`).
4. **Transcription & translation** — Runs Whisper in `translate` mode (auto-detects source language, outputs English) with voice-activity-detection (VAD) filtering enabled, and writes the result to `subtitles.en.srt`.
5. **Subtitle retiming** — This is the notebook's most involved step:
   - Extracts a mono 16kHz WAV from the audio.
   - Computes frame-level RMS energy and derives an adaptive threshold to detect speech vs. silence.
   - Closes small gaps and removes short noise bursts to produce clean speech intervals.
   - Aligns each subtitle line's original timing against these detected speech intervals, splitting a subtitle into multiple lines (proportionally, by sentence) if it spans more than one speech interval.
   - Enforces a minimum subtitle duration and a small gap between consecutive lines to avoid overlaps.
   - Writes the result to `subtitles.en.retimed.srt`.
6. **Download** — Provides a helper to download the most recently generated `.srt` file (or one matching a name filter) via the Colab file download dialog.

## Requirements

- Google Colab (uses `google.colab.files` for upload/download; a GPU runtime is recommended but not required).
- Python 3.12 (as run in the original notebook).
- Packages: `faster-whisper`, `ffmpeg-python`, `torch`, `numpy` (installed/assumed available in the Colab environment).
- `ffmpeg` binary (installed via `apt` in the first cell).

## Usage

Run the cells in order:

1. Run the setup cell to install dependencies and confirm GPU/CPU status.
2. Run the upload cell and select your video or audio file.
3. Run the model-loading cell (this downloads the `large-v3` Whisper model — may take a few minutes on first run).
4. Run the transcription/translation cell to produce `subtitles.en.srt`.
5. Run the retiming cell to produce `subtitles.en.retimed.srt`.
6. Run the download cell to save the desired `.srt` file to your machine. Pass a substring to `download_srt("retimed")` to specifically grab the retimed version, or call it with no arguments to grab whichever `.srt` was most recently written.

## Output files

| File | Description |
|---|---|
| `audio.m4a` | Extracted audio (only created if the input was a video file) |
| `subtitles.en.srt` | Raw English subtitles from Whisper, using Whisper's own segment timestamps |
| `subtitles.en.retimed.srt` | Subtitles re-aligned to detected speech intervals, with long segments split across multiple speech bursts |

## Notes / tuning

Key parameters live near the top of the retiming cell and can be adjusted:

- `GAP_SEC` (0.04s) — minimum gap enforced between consecutive subtitle lines.
- `MIN_DUR` (0.25s) — minimum duration for any subtitle line.
- `VAD_MIN_SIL_MS` (900ms) — minimum silence length that counts as a genuine gap between speech bursts (shorter gaps get merged).
- `VAD_MIN_SPEECH_MS` (180ms) — minimum length for a detected speech burst; anything shorter is discarded as noise.
- `MERGE_GAP_SEC` (0.06s) — speech intervals closer together than this are merged into one.

The energy-based VAD in the retiming step is a custom lightweight implementation (frame RMS + adaptive thresholding), separate from the VAD filter used inside Whisper's own transcription call.
