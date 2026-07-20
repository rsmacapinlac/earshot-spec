# Non-Functional Requirements

## NFR-1: Core offline operation
Recording, finalization, and local transcription all function offline; the web UI is
served on the LAN and needs no internet. **Diarization (FR-25) is the one optional
feature that requires internet** (OpenAI). The device is fully functional without it.

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
- Summarization of transcripts (designed-for; exposed via the web UI later)
- Server-side transcription **as the primary transcript path** — the local
  faster-whisper transcript stays authoritative; OpenAI is used only for the optional
  diarized transcript (FR-25).
