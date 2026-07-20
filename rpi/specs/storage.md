# Storage

Local file storage, filesystem-as-state, disk management, and crash recovery
(FR-7). The stored artifact is a single WAV ([ADR-0001](../adr/0001-audio-storage-format.md)).

## FR-7: Local storage
- Recordings are saved locally and stay on the device until offloaded. 
- Path: `<storage.data_dir>/recordings/<YYYYMMDDTHHMMSS>/` (default
  `~/earshot/recordings/…`, e.g. `20260717T114313`).
- The session directory holds, across its lifecycle:

| File | When | Notes |
|---|---|---|
| `recording-NNN.wav` (×N) | during recording, transiently | 16 kHz stereo PCM chunks; **deleted after `session.wav` is written** |
| `session.wav` | at session end | concatenated single WAV — the retained offload/transcription artifact |
| `status.json` | at session end | `earshot-tui` status mirror (see below) |
| `transcript.md` | after transcription | `earshot-tui`-compatible transcript |
| `transcript_raw.json` | after transcription | raw faster-whisper segment data |

## Filesystem as state
The filesystem is the source of truth — no database ([ADR-0006](../adr/0006-filesystem-as-state.md)).

| Session directory contents | Meaning |
|---|---|
| `recording-NNN.wav` only (no `session.wav`) | Recording in progress, or crashed before concatenation |
| `session.wav`, no `transcript.md` | Finalized; **pending transcription** |
| `session.wav` + `transcript.md` | Fully processed |

`status.json` mirrors the derived state for `earshot-tui` but is **not**
authoritative:
```json
{ "status": "encoded" | "transcribed",
  "device": "pi-earshot-pi4",
  "recorded_at": "2026-07-17T12:28:13.601387",
  "duration": 2604.8065,
  "transcribed_at": "2026-07-17T13:01:02.335633" }
```
> `"encoded"` is a legacy status literal kept for `earshot-tui` compatibility; it
> means capture was finalized to `session.wav` (no encode occurs).

## Disk space management
- Disk usage is checked before each new recording and continuously while blocked.
- At or above `storage.disk_threshold_percent` (default 90%), the LED pulses
  **orange** and new recordings are blocked.
- The device recovers automatically once files are removed and usage drops below
  the threshold.
- A WAV session is large (~3.8 MB/min stereo), so this threshold is the primary
  storage guard — offload frequently.

## Crash recovery
On startup, orphaned sessions are finalized (NFR-2). For each session directory:

- If `session.wav` exists, the session is already finalized — skip (it is pending
  transcription if it has no `transcript.md`).
- Otherwise, if `recording-NNN.wav` chunks are present (a crash before
  concatenation), **concatenate** them into `session.wav`, then delete the chunks —
  the same end-of-session path as [recording.md](recording.md#fr-3--fr-6-end-of-session--concatenate-to-one-wav).
  - A chunk WAV whose header was never finalized (crash before `close()`) is read
    tolerantly (frame count from file size).
  - If concatenation fails (e.g. no disk space), the chunks are **left in place**
    and logged; recovery is retried on the next boot. A single failure never aborts
    the scan of other sessions.
