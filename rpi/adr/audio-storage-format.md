# Audio Stored as a Single m4a File

**Status:** Accepted 

## Context

A session is captured as chunked 16 kHz/16-bit mono PCM WAV (see [Chunked recording](chunked-recording.md)). The stored and transcribed artifact needs a single defined format.

Uncompressed PCM is 256 kbps ≈ 1.9 MB/min — ~83 MB for a 43-minute session. The 90% disk threshold is the device's primary storage guard, and it *blocks new recordings* rather than pruning, so on a 59 GB card the user must intervene after roughly 28 hours of audio.

## Decision

Audio is stored as a single **`session.m4a`** — **AAC-LC, 32 kbps, 16 kHz mono**.

- **32 kbps** is the working point for 16 kHz mono speech: ~8× smaller than PCM on
  material that is voice-grade to begin with. Configurable via
  `recording.encode_bitrate_kbps`.
- **Container and codec are fixed**, not configurable. A format switch changes what every
  downstream spec reads and would double the state space in storage, transcription, and
  recovery — an ADR decision, not a config flag.
- **Capture is unchanged and lossless.** Recording still writes PCM WAV chunks for
  in-session crash resilience ([chunked recording](chunked-recording.md)). Only the
  finalized artifact is compressed.
- **Finalization concatenates and encodes in a single `ffmpeg` pass** over the chunk list
  (concat demuxer → AAC), so no intermediate full-length WAV is written. Contract:
  [specs/recording.md](../specs/recording.md#fr-3--fr-6-end-of-session--encode-to-one-m4a).

## Consequences

- **~8× less storage.** A 43-minute session goes from ~83 MB to ~10 MB; a 59 GB card holds ~230 hours instead of ~28. The disk threshold stops being a routine concern.
- **Capture is lossless; the stored artifact is not.** This is the real cost. PCM exists only for the life of the session, and the encode is one-way — the original samples are gone once the chunks are deleted.
- **Both processing routes take the file as stored.** Local transcription decodes `m4a` directly; a service job submits it unchanged, with no transcode or split on the device — see [specs/processing.md](../specs/processing.md#two-routes-one-of-them-optional).
- **Finalization becomes CPU-bound**, where concatenation was I/O-bound. Encoding 16 kHz mono AAC runs many times faster than realtime, so the amber window stays short (estimated tens of seconds for a long session), but it is no longer a plain file copy.
- **Peak disk at finalization is ~1.13× session size** (chunks plus the growing m4a), versus ~2× had an intermediate WAV been written.
- **`ffmpeg` is genuinely required on the device**, for the encode. 
- **The `"encoded"` status literal became truthful.** It was retained for `earshot-tui` compatibility with a note that no encode actually occurred; now one does.
- Crash recovery finalizes an interrupted session by running the same encode on whatever chunks exist. See [storage.md](../specs/storage.md#crash-recovery).

## Alternatives

- **Uncompressed WAV** — the original decision. Lossless, no encoder, directly playable
  everywhere, and finalization was a single concatenation. Rejected on re-decision: its
  decisive "no encoder" argument no longer applies now that diarization needs one anyway,
  and it costs 8× the storage on a device whose primary storage guard is a hard block.
- **64 kbps AAC** — rejected as the default: doubles storage for headroom this source
  material does not need. Reachable via `recording.encode_bitrate_kbps` if measurement
  shows 32 kbps hurts accuracy.
- **24 kbps AAC** — rejected: near the floor where AAC audibly degrades speech, risking
  listen-back and transcription quality to save storage that is no longer scarce.
- **Encode each chunk, then stitch** — rejected: moves encoder work into live capture, and
  concatenating AAC streams cleanly is fiddly.
- **Encode directly from ALSA, no WAV at all** — rejected: a crash would lose the entire
  session, contradicting [chunked recording](chunked-recording.md) and the
  [resilience requirement](../requirements/non-functional/resilience.md). The WAV chunks
  are the crash guarantee.

