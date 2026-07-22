# 0001 — Processing as a Separate Containerised Service

**Status:** Accepted (2026-07-21)

## Context

The Raspberry Pi recorder ran transcription on-device with faster-whisper. Measured
against its own spec that cost 7–13 minutes per 15 minutes of audio on a Pi 4B, holding
two of four cores for the duration — which is why the device needed a rule cancelling
transcription whenever a recording started, and a whole LED state to represent "busy".

Diarization was worse. It went to OpenAI not because a cloud model was better, but
because nothing of the required quality would run on the hardware. That import brought
its own constraints: a 25-minute per-request cap, speaker labels uncorrelated across
requests, an API key to store and protect, per-session cost, and the product's only
dependency on the internet.

Every one of those problems traces to the same root: **a Pi 4B is a good recorder and a
poor inference host.** The device was being asked to do work it is unsuited for, and the
architecture was accumulating workarounds for that mismatch.

## Decision

Processing may run in a **separate service, deployed with Docker Compose**, exposing an
HTTP API for transcription and diarization.

The service is **optional**. Devices remain self-sufficient — the Pi transcribes locally
by default ([rpi ADR-0010](../../rpi/adr/0010-optional-processing-service.md)) — and this
service is what they use *instead*, when one is available, plus the only route to
diarization.

The service is **device-agnostic** — it takes audio and returns segments, with no notion
of sessions, recording IDs, or users. It is specified as its own track rather than as
part of the Pi's, because it has its own deployable lifecycle and its own implementation
repository, and because the ESP32 recorder's v1 transfer seam is a designed-for path to
the same API.

## Consequences

- **A device using the service does no inference**, so a service job never contends with
  capture and is never preempted by a recording. The device keeps its local path and its
  preemption rule for when no service is configured.
- **Diarization can use open-source models** ([ADR-0003](0003-open-source-diarization.md)),
  because the host is chosen for the job. That removes the 25-minute cap, the split, the
  duplicate speaker labels, the API key, and the per-session cost in one move.
- **earshot becomes fully local-first.** With processing on the LAN, no feature requires
  the internet.
- **Cost: a second thing to run** — a container, a volume of weights, a host to keep
  alive. This is why it is optional. A user who does not want that burden simply does not
  deploy it, and keeps a working recorder.
- **Cost: two processing paths on the device**, with two failure modes and a routing rule.
  Accepted as the price of not forcing a deployment on anyone.
- **The device must tolerate the service being absent.** Without a configured URL, or with
  the service unreachable, the recorder records, stores, plays, serves its UI, and
  transcribes locally — it simply offers no diarization. Degradation, not failure.

## Alternatives

- **Keep all processing on the device** — self-contained, no second deployment. Not
  rejected for transcription, which remains the device default. Rejected for diarization
  only: no amount of tuning makes a Pi 4B a reasonable diarization host.
- **Make the service mandatory** — one processing path instead of two. Rejected: it turns
  a plug-in appliance into a two-component system, which is a worse product for anyone
  without a homelab.
- **A cloud API for everything** — no hardware to run at all. Rejected: it contradicts the
  product's local-first posture, sends private conversation audio to a third party by
  default, and makes the recorder useless without internet.
- **Specify the service inside the `rpi/` track** — fewer moving parts. Rejected: it is
  not a Pi concern, it deploys and versions separately, and coupling it to one device's
  docs would obstruct the ESP32 track using it later.
- **A library the device imports rather than a service** — no network hop. Rejected: it
  does not solve the problem, which is that the device lacks the compute. The work has to
  happen elsewhere, and "elsewhere" means a network boundary.
