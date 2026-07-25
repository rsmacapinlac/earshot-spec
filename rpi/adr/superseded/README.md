# Superseded ADRs (RPi)

Replaced decisions, kept for history and **no longer normative**. Each carries a
`Superseded` status line pointing to the record that replaced it.

These came from the retired `service/` track, written when earshot planned to build its own
processing service. All are superseded by
[adopt an off-the-shelf processing service](../off-the-shelf-processing-service.md).

| Decision | Note |
|---|---|
| [Processing as a separate containerised service](separate-processing-service.md) | Off-device/optional stands, now in [optional processing service](../optional-processing-service.md) |
| [Open-source diarization (pyannote)](open-source-diarization.md) | Now a selection criterion, not a build spec |
| [Asynchronous job API](async-job-api.md) | The adopted service is synchronous; the device owns queuing |

Current decisions are in [`../`](../README.md).
