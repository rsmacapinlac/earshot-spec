# 0001 — Audio Stored as a Single WAV File

**Status:** Accepted

## Context

Sessions are captured as chunked 16 kHz PCM WAV ([ADR-0007](0007-chunked-recording.md)).
The stored, offloaded, and transcribed artifact needs a single defined format.

## Decision

Audio is stored as **WAV**. At end of session the `recording-NNN.wav` chunks are
concatenated into one **`session.wav`** (same-format PCM concat, no re-encode);
`session.wav` is retained and the chunks are deleted. There is no compression or
encode stage. Transcription and USB offload both operate on `session.wav`.

Rationale:
- **Lossless** — the captured PCM is preserved exactly; no generational loss.
- **No encoder** — no codec dependency to install, no bitrate/quality tuning
  surface to get wrong.
- **Directly usable** — the offloaded file plays and edits everywhere with no
  decoder on the receiving machine.
- **Simplest path** — end-of-session work is a single concatenation.

## Consequences

- WAV is large: ~3.8 MB/min at 16 kHz stereo (≈165 MB for a 43-min session). The
  disk-threshold block (default 90%) is the primary storage guard — offload
  frequently. See [storage.md](../specs/storage.md#disk-space-management).
- The post-recording (amber) window is **I/O-bound** — concatenating a large WAV,
  not CPU-bound encoding — so stopping a session is fast.
- **`ffmpeg` is still required**: faster-whisper uses it to decode `session.wav`
  for transcription. Concatenation and duration use the Python `wave` module.
- Crash recovery finalizes an interrupted session by **concatenation** on boot —
  no re-encode, no mono/stereo asymmetry, no failure marker. See
  [storage.md](../specs/storage.md#crash-recovery).
- USB offload moves `session.wav`; FAT32's 4 GB per-file limit is comfortably
  above a single session.
- Whether the stored WAV should be stereo or mono (mono halves size at no
  speech-to-text cost) is
  [TD-2](../requirements/open-technical-decisions.md#td-2--stored-wav-stereo-vs-mono).
- **Implementation gap:** the running v0.2.2 app still encodes to `session.opus`;
  it must be updated to store `session.wav` only to match this spec.
