# earshot processing service — Specs Index

Normative behavior: the HTTP contract, the processing pipeline, and the deployment
surface. These specs define the **target behavior** the implementation must meet; they
are authoritative when other docs disagree.

Product needs and scope: [`../requirements/`](../requirements/README.md).
Rationale: [`../adr/`](../adr/README.md).

## Documents

| File | Covers |
|---|---|
| [api.md](api.md) | HTTP contract — job lifecycle, payloads, result shape, retention (SR-1–SR-7) |
| [processing.md](processing.md) | Decode, transcribe, diarize, speaker assignment, models, failure codes (SR-2, SR-3) |
| [deployment.md](deployment.md) | Compose file, environment variables, volumes, host requirements (SR-10) |

## Conventions

- **SR-n** are stable service requirement IDs, defined in
  [`../requirements/README.md`](../requirements/README.md). **SNFR-n** are the
  non-functional ones. IDs are never reused; retired requirements leave gaps.
- The service returns **segments, never rendered text**. Any Markdown, header, or
  timestamp formatting is the calling device's concern — see
  [ADR-0002](../adr/0002-async-job-api.md).
- The service is **stateless about devices**. It has no notion of a session, a recording
  ID, or a user; it takes audio and returns segments. That is what lets one deployment
  serve both device tracks.
