# Adopt an Off-the-Shelf Processing Service, Don't Build One

**Status:** Accepted (2026-07-25)

## Context

[Optional processing service](optional-processing-service.md) established that a processing
service is an optional upgrade — faster transcription, and the only route to diarization.
An earlier draft went further and specified earshot's *own* bespoke service: a whole
`service/` documentation track with a custom `/v1` asynchronous job API, its own container
image, and requirements describing software earshot would build, version, and maintain.

Building it proved unjustified. Capable off-the-shelf ASR + diarization services already
exist, run locally in a container, and do exactly what those requirements asked for.
Writing our own duplicates working software and adds a second codebase to release — at odds
with the principle that the optional upgrade stay lightweight.

What earshot actually needs from a processing service is small and stable:

- runs on the operator's own LAN, with **no cloud dependency** (local-first);
- **transcribes**, and does **speaker diarization consistent across a whole recording** —
  the property a chunked cloud API cannot give, and the reason diarization cannot just be a
  third-party API call;
- returns **structured segments** (the device renders the transcript);
- **deploys with Docker**.

## Decision

**earshot does not build a processing service. It adopts an existing third-party one** that
meets the needs above, deployed by the operator on the LAN.

The device integrates against **whatever contract that service exposes** — earshot does not
design it. earshot owns only:

1. the **selection criteria** (this ADR) and the **evaluation** of a specific candidate
   (an [experiment](../experiments/0002-whisper-asr-webservice.md));
2. its **device-side client** — routing, queuing, retry, and normalizing the service's
   output into earshot's `Segment` shape and `Speaker N` labels
   ([processing.md](../specs/processing.md#fr-15b-process--service));
3. a **deployment recipe** for the chosen service
   ([reference](../reference/processing-service.md)).

The specific service in use is chosen and validated by experiment and is **swappable
without revisiting this decision**. The first adopted service is
`ahmetoner/whisper-asr-webservice` (WhisperX) —
[experiment 0002](../experiments/0002-whisper-asr-webservice.md).

## Consequences

- **No bespoke service to build, version, or run.** The `service/` spec track is retired;
  its records are superseded by this one (see [ADR index](README.md#superseded)).
- **The contract is taken as given.** The adopted service may be synchronous, may offer no
  jobs/progress/cancel, and may name speakers its own way. Anything the device wants that
  the service does not provide is the **device's** responsibility — queuing, progress,
  retry, timeout, and speaker-label normalization all live on the Pi
  ([processing.md](../specs/processing.md#fr-15b-process--service)).
- **Capabilities are discovered, not dictated.** The device probes the service to learn
  whether diarization is offered, rather than reading a health contract earshot defined.
- **Swapping engines is an experiment, not an ADR change.** A better service is evaluated
  against the same criteria; this decision stands.
- **Local-first is preserved.** The criteria forbid a cloud dependency, so adopting
  third-party software still keeps audio on the LAN.
- **Cost: earshot's behavior is bounded by software it does not control.** Accuracy,
  performance, and API shape are inputs, not choices. If a service stops meeting the
  criteria, earshot adopts another.

## Alternatives

- **Build a bespoke service** (the retired `service/` track) — full control of the API
  (async jobs, progress, cancel). Rejected: it duplicates capable existing software and
  adds a second codebase to maintain for an *optional* upgrade, buying control the device
  can supply itself — it already owns a durable job queue
  ([job execution](job-execution.md)). The asynchronous job API this enabled is
  [superseded](superseded/async-job-api.md).
- **A cloud API for processing** — nothing to run. Rejected: it contradicts local-first and
  reintroduces the cloud-diarization constraints (per-request caps, cross-request label
  drift, an API key, per-session cost, an internet dependency) that
  [open-source diarization](superseded/open-source-diarization.md) removed.
- **A library the device imports** — no network hop. Rejected: the device lacks the
  compute; the work must happen on another host, which means a network boundary regardless.
