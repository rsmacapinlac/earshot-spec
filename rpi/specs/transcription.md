# Transcription

Opt-in on-device transcription with faster-whisper (CTranslate2). Runs during
idle time only; never competes with recording. LED behavior:
[state-machine.md](state-machine.md).

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

### FR-14a: Scheduling
- Transcription runs only when idle **and** the queue is non-empty. The idle loop
  arms a timer and enters transcription after **~180 s** of idle (re-armed each
  check while the queue stays empty).
- Starting a recording cancels in-progress transcription immediately; the session
  returns to the **front** of the queue and recording takes priority.
- On the next return to idle (after finalizing), the queue is re-checked and
  transcription resumes from the front.
- On boot, pending sessions are picked up once the device reaches idle.

## FR-15: Process
- The model is loaded once per queue run:
  `WhisperModel(model, device="cpu", download_root="~/.local/share/earshot/models", cpu_threads=threads)`.
  A load failure aborts the run (retried next idle window).
- Each pending session's **`session.wav`** is transcribed (`transcribe_session`);
  faster-whisper decodes the WAV via ffmpeg and reads lazily during segment
  iteration.
- Runs in a cancellable worker thread. Cancellation trigger:
  - **Button press** → return `"button"` (start recording).
- On success: write `transcript_raw.json` then `transcript.md`, delete any
  `.transcription_failures.json` retry-state file, update `status.json` to
  `transcribed` (+ `transcribed_at`), pop the session, and re-scan for newly
  arrived sessions.
- On failure: no transcript is written; the failure is logged with the session
  path and attempt count. Increment the persisted failure count in
  `.transcription_failures.json` so retries survive service restarts and reboots.
  If `transcription.max_failures = 0`, the session stays at the front of the queue
  and is retried on the next idle window forever. If `max_failures > 0`, retry
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
- **Amber**, very slow pulse (~1.5–2 s) while transcribing — distinct from
  warning orange.
- Returns to solid **green** when the queue empties (no transition animation).
- On the ReSpeaker HAT, the LED is the only transcription feedback.

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
is idle-only, so the longer `base.en` runtime does not affect recording.
