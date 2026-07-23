# Chunked Recording (15-Minute Default)

**Status:** Accepted

## Context
A single session can run for hours. Writing audio continuously to one file means
a crash or power loss mid-session loses everything since the last stop.

## Decision
Within a session, audio is split into configurable chunks (default: 15 minutes).
Each chunk is a separate WAV file (`recording-001.wav`, `recording-002.wav`, …).
Chunk rollover is triggered by the timer only; the button stop closes the current
chunk and ends the session. Chunk duration is set via
`recording.chunk_duration_seconds`.

## Consequences
- Maximum data loss from a crash is one chunk duration (default 15 min), not the
  whole session.
- No hard maximum session duration — recording continues until the button is
  pressed or the disk threshold is reached.

> **Pipeline (see [Audio storage format](audio-storage-format.md)):** `recording-NNN.wav`
> chunks accumulate during recording for in-session crash resilience; at session
> end they are concatenated and encoded into a single **`session.m4a`** and then
> deleted. `session.m4a` is the retained artifact. See
> [specs/recording.md](../specs/recording.md).
