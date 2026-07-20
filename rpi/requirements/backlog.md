# Backlog

Out of scope for the current release; candidates for future work. Grouped by
theme, roughly ordered by likely priority.

## Audio feedback (v2)
Audio feedback for state transitions is deferred to v2. Requires an
`AudioOutputInterface` in the HAL ([ADR-0003](../adr/0003-hardware-abstraction-layer.md)).

| ID | Item | Notes |
|---|---|---|
| B-A1 | Audio cues on state transitions | Short tones on start/stop recording, USB complete, error. |
| B-A2 | Configurable audio feedback volume | Via an `[audio]` config key. |

## Transcription
| ID | Item | Notes |
|---|---|---|
| B-T1 | `storage.require_transcript_before_offload` | Block USB offload until pending transcription completes. Deferred: offload-regardless is the safer default. |
| B-T2 | Installer model-choice prompt | Default is `base.en`; a prompt could let users pick `tiny.en` for faster, lighter transcribes. |
| B-T3 | Transcription retry limit | After N failures, write a `.failed_transcription` marker and move on instead of retrying indefinitely. |
| B-T4 | Real-time / live transcription | Transcribe during recording. Significant complexity; out of scope until post-session transcription is stable. |

## Recording
| ID | Item | Notes |
|---|---|---|
| B-R1 | Wake-word detection | Auto-start on a trigger word. Always button-triggered in v1. |
| B-R2 | Multi-device coordination | Multiple devices recording one session with synced timestamps. No design work done. |

## Infrastructure / UX
| ID | Item | Notes |
|---|---|---|
| B-I1 | Web UI / local dashboard | Browser interface over WiFi to review recordings/transcripts without USB offload. A companion server is a natural fit for Docker (see [ADR-0002](../adr/0002-python-venv-over-docker.md)). |
| B-I2 | Pi 5 support | ~3–5× faster transcription than Pi 4B. Untested; likely straightforward. |
| B-I3 | WiFi onboarding without SSH | Hotspot or captive portal for configuring WiFi with no existing network. |
