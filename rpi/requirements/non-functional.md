# Non-Functional Requirements

## NFR-1: No network dependency
Recording, finalization, and transcription all function offline.

## NFR-2: Resilience
- A crash or power loss after recording must not lose the raw audio.
- A session finalization failure does not delete the chunk WAVs; recovery is
  retried later.

See [../specs/storage.md](../specs/storage.md#crash-recovery) for the exact
recovery contract.

## NFR-3: Startup time
| SBC | Target: power-on → green-light ready |
|---|---|
| Pi 4B | 60 s |

## Out of scope (v1)
- Real-time / live transcription during recording
- Speaker identification / diarization — out of scope for v1 because v1 stores mono audio and does not perform speaker embeddings or enrollment.
- Server-side transcription
