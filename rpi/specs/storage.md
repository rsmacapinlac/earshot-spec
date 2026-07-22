# Storage

Local file storage, filesystem-as-state, disk management, and crash recovery
(FR-7). The stored artifact is a single m4a (see
[Audio storage format](../adr/0001-audio-storage-format.md)).

## FR-7: Local storage
- Recordings are saved locally and stay on the device until manually removed.
- Path: `<storage.data_dir>/recordings/rec-NNNNNN/` (default
  `~/earshot/recordings/…`, e.g. `rec-000042`).

### Session identity
The directory name is the session's stable identity and carries **no date or time**
(see [Session identity](../adr/0008-session-identity.md)).

- IDs are `rec-` plus a zero-padded 6-digit decimal sequence, allocated monotonically:
  `rec-000001`, `rec-000002`, …
- Allocation is `max + 1` over the existing `rec-*` directories. Because only one session
  is ever active (FR-23), allocation cannot race.
- A recoverable allocator file `recordings/.next_id` may be kept to avoid reusing an ID
  after a delete. It is **not a manifest**: if it is missing, unreadable, or lower than
  the scan result, recover by scanning and taking `max + 1`. The directory scan is always
  authoritative.
- Zero-padding keeps lexical order equal to numeric order, so a plain directory sort is
  creation order.
- Allocating an ID requires no clock, so a session started on a device that has never had
  a valid time is indistinguishable from any other.
- The session directory holds, across its lifecycle:

| File | When | Notes |
|---|---|---|
| `recording-NNN.wav` (×N) | during recording, transiently | 16 kHz mono PCM chunks; **deleted after `session.m4a` is written** |
| `session.m4a` | at session end | AAC-LC 32 kbps — the retained artifact for playback, download, transcription, and diarization upload |
| `status.json` | at session end | `earshot-tui` status mirror (see below) |
| `transcript.md` | after a completed job | The single `earshot-tui`-compatible transcript, rendered by the device from the service's segments. A diarize job **overwrites** it with the speaker-labelled version |
| `transcript_raw.json` | after a transcribe job | raw segments returned by the processing service |
| `transcript_diarized_raw.json` | after a diarize job (optional) | raw speaker-labelled segments; its presence marks the current `transcript.md` as the diarized version (FR-20). Does not affect pending/failed state |
| `session.json` | when the user names the session or a speaker (optional) | **User-authored labels** — the session name (FR-29) and the `Speaker N` → name map (FR-27). Absent until something is named; see below |
| `.job.json` | while a job is in flight | `{ job_id, kind, submitted_at }` — lets the device resume polling after a reboot instead of resubmitting ([processing.md](processing.md#crash-resilience)). Deleted on completion |
| `.processing_failures.json` | after a failed processing attempt | persisted retry count and last error; deleted after success |
| `.failed_processing` | after repeated processing failure | retry-suppression marker; the web UI's **Retry** action (FR-24) deletes it, or delete it manually |

## Filesystem as state
The filesystem is the source of truth — no database (see
[Filesystem as state](../adr/0006-filesystem-as-state.md)).

| Session directory contents | Meaning |
|---|---|
| `recording-NNN.wav` only (no `session.m4a`) | Recording in progress, or crashed before the encode |
| `session.m4a`, no `transcript.md`, no `.failed_processing` | Finalized; **pending processing** |
| `session.m4a` + `transcript.md`, no `transcript_diarized_raw.json` | Transcribed |
| `session.m4a` + `transcript.md` + `transcript_diarized_raw.json` | Transcribed and **diarized** — `transcript.md` is the speaker-labelled version |
| `session.m4a` + `.failed_processing`, no `transcript.md` | Finalized; processing failed and is **not pending** until the marker is removed |

## User-authored labels — `session.json`

The directory name is the session's **identity**; everything a user types is a **label**
on top of it, and labels never participate in identity, ordering, or lookup. Those
labels are the one thing in a session directory that cannot be re-derived from anything
else, so they live in their own file rather than in the rebuildable `status.json` mirror:

```json
{ "name": "Weekly sync — pricing",
  "speakers": { "Speaker 1": "Ritchie", "Speaker 2": "Sarah" } }
```

- Both keys are optional; an absent or unparseable `session.json` means "unnamed" and is
  never an error. A `Speaker N` with no entry stays `Speaker N`.
- Names are **not unique** — two sessions may share a name. Nothing looks a session up
  by name.
- Deleting `session.json` reverts the session to its directory name and generic speaker
  labels; no other state is affected.

`status.json` mirrors the derived state for `earshot-tui` but is **not**
authoritative:
```json
{ "status": "encoded" | "transcribed" | "diarized",
  "device": "earshot-pi",
  "recorded_at": "2026-07-17T12:28:13.601387",
  "duration": 2604.8065,
  "transcribed_at": "2026-07-17T13:01:02.335633" }
```
> The `"encoded"` literal is chosen for `earshot-tui` compatibility; it means
> capture was finalized to `session.m4a`. It now describes a real encode stage.

### Time is metadata, not identity
`recorded_at` records **when** a session was captured; it never establishes **which**
session it is. Nothing sorts, looks up, or recovers by it.

- `recorded_at` is the wall-clock time at session start, captured in memory and written
  at finalization. The Pi 4B has no RTC ([hardware.md](../requirements/hardware.md)), so
  on a device that booted without reaching a time source it is a best guess.
- It stays a **mirror**, not an owned fact: if `status.json` is lost or rebuilt,
  `recorded_at` is re-derived from the session directory's creation time. The same
  fallback covers crash recovery, where the in-memory start time is gone.
- A session with no trustworthy time is fully usable: it has an ID, it can be named,
  ordered, played, transcribed, and deleted. Only the displayed date is degraded, and
  nothing reads it back.

## Disk space management
- Disk usage is checked before each new recording and continuously while blocked.
- At or above `storage.disk_threshold_percent` (default 90%), the LED pulses
  **orange** and new recordings are blocked.
- The device recovers automatically once files are removed and usage drops below
  the threshold.
- A finished session is ~0.24 MB/min (AAC 32 kbps), so a 59 GB card holds roughly
  230 hours. The threshold remains the storage guard, but it is no longer a routine
  concern. Note the transient peak: during a session the uncompressed chunks are on disk
  at ~1.9 MB/min until the encode replaces them.

## Crash recovery
On startup, orphaned sessions are finalized (NFR-2). For each session directory:

- If `session.m4a` exists, the session is already finalized — skip (it is pending
  processing only if it has no `transcript.md` and no `.failed_processing`). If it carries
  a `.job.json`, resume polling that job ([processing.md](processing.md#crash-resilience)).
- Otherwise, if `recording-NNN.wav` chunks are present (a crash before
  the encode), run the **same concatenate-and-encode pass** to produce `session.m4a`,
  then delete the chunks — the end-of-session path in
  [recording.md](recording.md#fr-3--fr-6-end-of-session--encode-to-one-m4a).
  - A chunk WAV whose header was never finalized (crash before `close()`) is read
    tolerantly (frame count from file size).
  - If the encode fails (e.g. no disk space), the chunks are **left in place**
    and logged; recovery is retried on the next boot. A single failure never aborts
    the scan of other sessions.
