# 0001 — Audio Stored as a Single WAV File

**Status:** Accepted

## Context

Sessions are captured as chunked 16 kHz PCM WAV (see
[Chunked recording](0007-chunked-recording.md)). The stored and transcribed
artifact needs a single defined format.

## Decision

Audio is stored as **WAV**. At end of session the `recording-NNN.wav` chunks are
concatenated into one **`session.wav`** (same-format PCM concat, no re-encode);
`session.wav` is retained and the chunks are deleted. There is no compression or
encode stage. Transcription operates on `session.wav`.

Rationale:
- **Lossless** — the captured PCM is preserved exactly; no generational loss.
- **No encoder** — no codec dependency to install, no bitrate/quality tuning
  surface to get wrong.
- **Directly usable** — `session.wav` plays and edits everywhere with no special
  decoder.
- **Simplest path** — end-of-session work is a single concatenation.

## Consequences

- WAV is large: ~1.9 MB/min at 16 kHz mono (≈83 MB for a 43-min session). The
  disk-threshold block (default 90%) is the primary storage guard — remove or
  archive recordings regularly. See
  [storage.md](../specs/storage.md#disk-space-management).
- The post-recording (amber) window is **I/O-bound** — concatenating a large WAV,
  not CPU-bound encoding — so stopping a session is fast.
- **`ffmpeg` is still required**: faster-whisper uses it to decode `session.wav`
  for transcription. Concatenation and duration use the Python `wave` module.
- Crash recovery finalizes an interrupted session by **concatenation** on boot —
  no re-encode, no mono/stereo asymmetry, no failure marker. See
  [storage.md](../specs/storage.md#crash-recovery).
- The WAV is **mono** (the left mic); see
  [recording.md](../specs/recording.md#capture-spec).
