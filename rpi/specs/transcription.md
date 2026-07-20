# Transcription

Opt-in on-device transcription with faster-whisper (CTranslate2). Runs during
idle time only; never competes with recording. LED behavior:
[state-machine.md](state-machine.md).

> **Engine.** faster-whisper, imported as a Python library
> (`from faster_whisper import WhisperModel`). Transcription reads **`session.wav`**
> ([ADR-0001](../adr/0001-audio-storage-format.md)).

## FR-14: Queue
- Enabled unless `transcription.enabled = false`.
- A session is **pending** when its directory has `session.wav` but no
  `transcript.md`.
- The queue is implicit — derived from the filesystem at runtime (no queue file).
- Processed **FIFO**, oldest session directory first (by timestamp name).
- The queue persists across reboots; a session stays pending until `transcript.md`
  is written.

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
- Runs in a cancellable worker thread. Cancellation triggers:
  - **Button press** → return `"button"` (start recording).
  - **USB stick inserted** → return `"usb"` (offload).
- On success: write `transcript_raw.json` then `transcript.md`, update
  `status.json` to `transcribed` (+ `transcribed_at`), pop the session, and
  re-scan for newly arrived sessions.
- On failure: no transcript is written; the session stays at the front of the
  queue; the failure is logged; the run ends and is retried on the next idle
  window.

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
- **Duration:** total audio duration across all chunks.
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
- Writes `transcription.enabled` and `transcription.model` to `config.toml`.
- `--no-transcription` skips the model download; users can also set
  `enabled = false` post-install.

## Performance
| Model | 15-min session (Pi 4B) |
|---|---|
| `base.en` (default) | ~7–13 min |
| `tiny.en` (lighter) | ~3–6 min |

Default thread count is 2 (headroom for recording on the 4-core CPU). Transcription
is idle-only, so the longer `base.en` runtime does not affect recording.
