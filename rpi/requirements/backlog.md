# Backlog

Out of scope for the current release; candidates for future work. IDs are stable.

## Next release
The planned next focus.

| ID | Item | Notes |
|---|---|---|
| B-I1 | Web UI / local dashboard | Browser interface over WiFi to review recordings/transcripts without USB offload. A companion server is a natural fit for Docker (see [Python venv over Docker](../adr/0002-python-venv-over-docker.md)). |

## Transcription
| ID | Item | Notes |
|---|---|---|
| B-T1 | `storage.require_transcript_before_offload` | Block USB offload until pending transcription completes. Deferred: offload-regardless is the safer default. |
| B-T3 | Transcription retry limit | After N failures, write a `.failed_transcription` marker and move on instead of retrying indefinitely. |
| B-T4 | Real-time / live transcription | Transcribe during recording. Significant complexity; out of scope until post-session transcription is stable. |

