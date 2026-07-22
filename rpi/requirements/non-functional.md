# Non-Functional Requirements

## NFR-1: Standalone, and no internet dependency
**The device is fully functional on its own.** Recording, finalization, storage, playback,
the web UI **and transcription** all run on the Pi with no service, no account, and no
internet ([ADR-0010](../adr/0010-optional-processing-service.md)).

**No earshot feature requires the internet.** An optional
[processing service](../../service/README.md) on the LAN makes transcription much faster
and adds diarization — the one capability the Pi cannot provide — but it is an upgrade,
not a dependency. Remove it and the device falls back to local transcription.

Capture must never depend on anything external: recording works with no network at all,
and audio waits safely on the device until processing runs.

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
- **Local diarization.** Speaker labelling needs compute a Pi 4B does not have; it
  requires a processing service or it is not offered.
- Any cloud/third-party processing. Audio never leaves the local network, and no API key
  exists anywhere in the system.
