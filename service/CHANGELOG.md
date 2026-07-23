# Changelog

All notable changes to the earshot **processing service documentation** (specs,
requirements, ADRs). The documentation set is versioned as a whole; the current version
is recorded in the [root README](../README.md). Dates are ISO-8601 (YYYY-MM-DD).

## [Unreleased]

### Added

- **Initial processing-service documentation set** — the target specification for a
  containerised transcription and diarization service (not yet built). Structured to
  mirror the device tracks: `requirements/`, `specs/`, `adr/`.
- **Requirements** (`requirements/README.md`): capabilities `SR-1`–`SR-10` and
  non-functional targets `SNFR-1`–`SNFR-5` — no internet dependency, long recordings as
  the normal case, honest progress, degraded-not-broken when diarization is unavailable,
  and bounded concurrency.
- **Specs** (`specs/`): the asynchronous HTTP job contract (`api.md`), the decode →
  transcribe → diarize → assign pipeline with model defaults and failure codes
  (`processing.md`), and the Compose file, environment variables, volumes, and host
  requirements (`deployment.md`).
- **ADRs** (`adr/`): processing as a separate containerised service; the asynchronous
  job API returning segments rather than rendered text; and open-source diarization via
  pyannote instead of a cloud API.

### Decisions baked into the spec

- **The service is device-agnostic.** It takes audio and returns segments, with no notion
  of sessions, recording IDs, or users — which is what lets one deployment serve both the
  RPi and (later) ESP32 tracks.
- **Segments, never rendered text.** Transcript formatting, speaker names, and headers
  belong to the calling device, which has the context the service lacks.
- **Audio is deleted when a job reaches a terminal state** (SR-8). The service is not an
  archive; the device owns the durable copy.
- **No authentication in v1** — trusted-LAN deployment, matching the device tracks.
- **A Raspberry Pi is not a supported host.** The split exists precisely because a
  recorder is poorly suited to this work — which is also why the service is optional
  rather than required.

### Relationship to the RPi track

This service is **optional**. It replaces the OpenAI diarization path outright, and
*offers an alternative to* on-device transcription without removing it — the Pi still
transcribes locally by default
([rpi: optional processing service](../rpi/adr/optional-processing-service.md)). Diarization is the one
capability that exists only here, because it needs compute a recorder does not have.

The open-source-diarization ADR delivers the RPi backlog item **B-T6** and closes that track's **TD-7** by
removing the constraint that produced it — the 25-minute per-request cap and speaker
labels uncorrelated across requests are properties of the cloud provider, not of the
problem.
