# Transcription

Opt-in on-device transcription with faster-whisper (CTranslate2). **Initiated from the
web UI** ([web-ui.md](../requirements/web-ui.md), FR-24); it never competes with
recording. LED behavior: [state-machine.md](state-machine.md).

> **Engine.** faster-whisper, imported as a Python library
> (`from faster_whisper import WhisperModel`). Transcription reads **`session.wav`**
> (see [Audio storage format](../adr/0001-audio-storage-format.md)).

## FR-14: Queue
- Enabled unless `transcription.enabled = false`.
- A session is **pending** when its directory has `session.wav` but no
  `transcript.md` and no `.failed_transcription` marker.
- The queue is implicit — derived from the filesystem at runtime (no queue file).
- Processed **FIFO**, oldest session directory first (by timestamp name).
- The queue persists across reboots; a session stays pending until `transcript.md`
  or `.failed_transcription` is written.

### FR-14a: Trigger
- Transcription is initiated per session from the web UI
  ([web-ui.md](../requirements/web-ui.md), FR-24).
- **Recording preempts transcription:** starting a recording (button or web, FR-23)
  cancels an in-progress transcription immediately and the session returns to the
  **front** of the queue, to be re-run from the web UI afterward.
- Pending sessions persist across reboots (FR-14) and are listed in the web UI for the
  user to run; a "transcribe all" action processes them FIFO.

## FR-15: Process
- The model is loaded once per queue run:
  `WhisperModel(model, device="cpu", download_root="~/.local/share/earshot/models", cpu_threads=threads)`.
  A load failure aborts the run (retried on the next transcription run).
- Each pending session's **`session.wav`** is transcribed (`transcribe_session`);
  faster-whisper decodes the WAV via ffmpeg and reads lazily during segment
  iteration.
- Runs in a cancellable worker thread. Cancellation trigger:
  - **A new recording** (button or web UI, FR-23) → cancel; the session returns to the
    front of the queue to be re-run later.
- On success: write `transcript_raw.json` then `transcript.md` (removing any
  `transcript_diarized_raw.json`, so re-transcribing a diarized session reverts it to a
  plain local transcript), delete any `.transcription_failures.json` retry-state file,
  update `status.json` to `transcribed` (+ `transcribed_at`), pop the session, and
  re-scan for newly arrived sessions.
- On failure: no transcript is written; the failure is logged with the session
  path and attempt count. Increment the persisted failure count in
  `.transcription_failures.json` so retries survive service restarts and reboots.
  If `transcription.max_failures = 0`, the session stays at the front of the queue
  and is retried on each subsequent transcription run indefinitely. If
  `max_failures > 0`, retry
  until the persisted count reaches `max_failures`; then write
  `.failed_transcription`, delete `.transcription_failures.json`, and skip the
  session on future queue scans until `.failed_transcription` is manually deleted.
- `.transcription_failures.json` contains at least `count`, `last_failed_at`, and
  `last_error`. `.failed_transcription` is a small text marker containing the
  failure count, timestamp of the last failure, and last error summary.
  `session.wav` remains in place for manual recovery or future retry;
  `transcript.md` is not written. Deleting `.failed_transcription` gives the
  session a clean retry attempt.

## FR-16: Transcript format
`transcript.md`, `earshot-tui`-compatible:
```markdown
# Recording — YYYY-MM-DD HH:MM:SS
**Duration:** Xh Xm Xs
**Processed:** YYYY-MM-DD HH:MM:SS

---

[MM:SS] segment text
[HH:MM:SS] segment text
```
- Header timestamp: the session directory name as a human-readable date.
- **Duration:** the `session.wav` duration (frame count ÷ sample rate).
- **Processed:** wall-clock time transcription completed.
- Timestamps use `[MM:SS]` under one hour, `[HH:MM:SS]` at/beyond one hour.
- Segment text is raw faster-whisper output — no post-processing.

`transcript_raw.json` accompanies it with the raw segment structures.

## FR-17: LED
- **Amber**, very slow pulse (~1.5–2 s) while a web-initiated transcription runs —
  distinct from warning orange.
- Returns to solid **green** when it completes (no transition animation).
- On the ReSpeaker HAT the LED is the local feedback; the web UI shows detailed
  progress and failures (FR-24).

## Diarization (FR-25)
Diarization is a **separate, web-initiated action** — not part of the local transcribe
path — and requires an OpenAI key. It sends the compressed session audio to OpenAI's
`gpt-4o-transcribe-diarize`, which transcribes and labels speakers in one pass, and
**overwrites** the session's `transcript.md` with that speaker-labelled transcript; the
raw response is saved to `transcript_diarized_raw.json` (see
[web-ui.md](../requirements/web-ui.md)). There is only ever one `transcript.md` per
session. Diarization does not require a prior local `transcript.md` — it can produce the
transcript on its own.

- Speakers are labelled from enrolled reference clips (FR-27) when available, else
  generic `Speaker N`.
- Sessions over OpenAI's per-request limits (25 MB / 1500 s) are compressed and split,
  reusing references to keep labels stable across parts. The exact chunking/stitching
  is **TD-7**, pending [experiment 0001](../experiments/0001-openai-diarization-mono-and-chunking.md).
- Requires network + a valid key; on failure the existing `transcript.md` (if any) and
  the audio are left intact — the overwrite happens only on success. Diarization does
  **not** affect the pending/failed state the transcribe queue derives from the
  filesystem.

## FR-18: Installer
- Installs `faster-whisper` via pip (no build step).
- Pre-downloads the configured model to `~/.local/share/earshot/models/`
  (default `base.en` INT8, ~60 MB).
- Writes `transcription.enabled`, `transcription.model`, and
  `transcription.max_failures` to `config.toml`; other `[transcription]` keys
  (e.g. `threads`) take their documented defaults.
- `--no-transcription` skips the model download; users can also set
  `enabled = false` post-install.

## Performance
| Model | 15-min session (Pi 4B) |
|---|---|
| `base.en` (default) | ~7–13 min |
| `tiny.en` (lighter) | ~3–6 min |

Default thread count is 2 (headroom for recording on the 4-core CPU). Transcription
is user-initiated and yields to recording (a new recording cancels it), so the longer
`base.en` runtime never blocks capture.
