# Backlog

Out of scope for the current release; candidates for future work. IDs are stable.

## Next release
The planned next focus.

_B-I1 (Web UI / local dashboard) has been **promoted** into the active requirements —
see [web-ui.md](web-ui.md). B-T6 (open-source / self-hosted diarization) has been
**promoted** and is delivered by the optional
[processing service](../../service/README.md) — diarization is available to users who run
one, and is not offered otherwise._

## Transcription
| ID | Item | Notes |
|---|---|---|
| B-T4 | Real-time / live transcription | Transcribe during recording. Significant complexity; out of scope until post-session transcription is stable. |
| B-T5 | Summarization | Post-session summary of a transcript; a designed-for web-UI action (see [web-ui.md](web-ui.md)), not built in v1. |

## B-T6 — Open-source / self-hosted diarization — **promoted**

_Promoted into the active design on 2026-07-21 and delivered by the
[processing service](../../service/README.md); see
[service ADR-0003](../../service/adr/0003-open-source-diarization.md) and
[rpi ADR-0010](../adr/0010-optional-processing-service.md). Retained below as the record of why._

v1 diarized via OpenAI `gpt-4o-transcribe-diarize` (FR-25). Its two per-request limits —
25 minutes, and no speaker-label correlation across requests — are what forced TD-7's
split-without-stitching resolution, and therefore why a long session lists more
`Speaker N` entries than there are people.

**The reframing that makes this tractable:** earshot already transcribes locally with
faster-whisper, so it does not need a service that transcribes *and* diarizes. It needs
only **speaker turns** — who spoke when — aligned onto segments faster-whisper has
already produced. That is a much smaller problem, and the one open source solves well.

Candidates: **pyannote.audio** (`speaker-diarization-3.1`, the reference toolkit — MIT
code, HuggingFace-gated models); **WhisperX** (packages faster-whisper + alignment +
pyannote); **sherpa-onnx** (ONNX Runtime rather than PyTorch, the lightest option and the
one most likely to fit this hardware).

Adopting any of them would diarize a whole recording in one pass with labels consistent
throughout, collapsing TD-7 entirely, and would make diarization local-first — which is
the repo's stated posture and the one place v1 departs from it.

**The catch is capacity.** pyannote's default pipeline pulls in PyTorch: a heavy install
and a large resident footprint, likely untenable on the 2 GB minimum and uncomfortable at
4 GB beside a CTranslate2 model, on a CPU without AVX. Two paths worth measuring before
committing — `sherpa-onnx` on the Pi itself, or hosting the diarizer on another LAN
machine so the Pi records and transcribes while a capable box handles turns on request.
The second keeps everything inside the network without asking a Pi 4B for work it suits
poorly.

This is **ADR-sized**, not a config change: it touches FR-25, FR-27, the API-key handling
(FR-26), NFR-1's network framing, and the installer. Re-verify the current state of these
projects and of OpenAI's limits before acting — if the duration cap has risen, part of the
motivation disappears.

