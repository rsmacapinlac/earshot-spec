# Non-Functional Requirements

## NFR-1: No network dependency
Recording, encoding, and transcription, all function offline.

## NFR-2: Resilience
- A crash or power loss after recording must not lose the raw audio.
- A single chunk encoding failure does not terminate the session — recording
  continues into the next chunk.

See [../specs/storage.md](../specs/storage.md#crash-recovery) for the exact
recovery contract.

## NFR-3: Startup time
| SBC | Target: power-on → green-light ready |
|---|---|
| Pi 4B | 60 s |

## Out of scope (v1)
- Wake-word detection (always button-triggered)
- Real-time / live transcription during recording
- Multi-device coordination
- Web UI or local dashboard
- Speaker identification / diarization — see Constraints
- Server-side transcription
- Audio feedback / speaker output (deferred to v2)
