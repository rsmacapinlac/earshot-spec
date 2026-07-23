# Non-Functional Requirements

Cross-cutting qualities the Raspberry Pi Earshot must have, as opposed to the features it
must provide. Each gets its own file, named for what it is — these are referred to by
name, not by number, so nothing has to be looked up in an index to be understood.

Functional requirements (`FR-n`) live in the sibling documents of
[`../`](../README.md); exact behaviour lives in [`../../specs/`](../../specs/README.md).

## Requirements

| Requirement | In short |
|---|---|
| [No internet; graceful degradation](no-internet-graceful-degradation.md) | Nothing needs the internet, capture needs no network at all, and everything else degrades by tier |
| [Clock independence](clock-independence.md) | Identity, ordering and labelling never depend on the clock being right — the Pi has no RTC |
| [Core functionality over the API](core-functionality-over-api.md) | Every core operation is reachable over the HTTP API; configuring the device stays SSH |
| [Resilience](resilience.md) | A crash or power loss never costs recorded audio |
| [Startup time](startup-time.md) | Power-on to ready in 60 s on a Pi 4B |

## Out of scope

- Real-time / live transcription during recording.
- Summarization of transcripts.
- **Local diarization.** Speaker labelling needs compute a Pi 4B does not have; it
  requires a [processing service](../../../service/README.md) or it is not offered.
