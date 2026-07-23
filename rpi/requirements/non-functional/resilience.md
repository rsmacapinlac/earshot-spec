# Resilience

## Requirement

- A crash or power loss **after recording** must not lose the raw audio.
- A session finalization failure does **not** delete the chunk WAVs; recovery is retried
  later.

Audio is the one artifact that cannot be regenerated. Transcripts can be re-run and
metadata rebuilt, so every failure path is designed to preserve capture at the expense of
anything else.

## How it is met

- Audio is written to **chunked WAV files during the session**, so an interruption costs
  at most one chunk rather than the whole recording
  ([chunked recording](../../adr/chunked-recording.md)).
- Finalization only deletes chunks **after** `session.m4a` is complete; a failed encode
  leaves them in place and removes any partial output.
- On startup, orphaned sessions are finalized by re-running the same encode. A single
  failure never aborts the scan of other sessions.
- A failed processing attempt never touches `session.m4a`.
- State lives in SQLite in WAL mode ([state storage](../../adr/state-storage.md)), so a
  power loss cannot leave a half-written session record — and the database is rebuildable
  from the files on disk, so losing it costs nothing permanent.

## Where this is specified

- [`specs/storage.md`](../../specs/storage.md#crash-recovery) — the exact recovery
  contract.
- [`specs/recording.md`](../../specs/recording.md#fr-6a-finalization-failure) —
  finalization failure behaviour.
