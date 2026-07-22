# 0010 — Processing Service Is Optional, Never Required

**Status:** Accepted (2026-07-21)

## Context

The Pi runs transcription locally with faster-whisper: 7–13 minutes per 15 minutes of
audio, holding two of four cores. That cost shapes real parts of the device's design — a
rule cancelling transcription when a recording starts, thread tuning, a model download in
the installer.

Diarization is worse. Nothing of the required quality runs on a Pi 4B at all. The previous
answer was OpenAI, which brought a 25-minute per-request cap, speaker labels uncorrelated
across requests, an API key to protect, per-session cost, and the product's only internet
dependency.

A [processing service](../../service/README.md) on the LAN removes all of that. The
question was whether the device should then *depend* on it.

It should not. **A desk recorder that requires you to stand up and maintain a container
somewhere is a different product** from one you plug in and use. That deployment burden
is not a detail — it decides whether earshot is an appliance or a component of a homelab.
An earlier draft of this decision made the service mandatory and was reversed for exactly
this reason.

## Decision

The device **stands alone**. With no configuration, no service, and no account, a Pi
records and transcribes. That baseline is inviolable.

A processing service is an **optional upgrade**. Setting `processing.service_url` routes
transcription to a machine better suited to it and unlocks diarization. Clearing it falls
back to local transcription; nothing breaks.

**Diarization has no local path and no cloud path.** It requires a service or it is not
offered. Reaching for a third-party API to provide it would trade the product's
independence for a feature — the same trade this decision exists to refuse.

## Consequences

- **The appliance property is preserved.** Plug in, press the button, get a transcript.
  No second thing to run, no account, no internet.
- **Local transcription is slow** — 20–37 minutes for a 43-minute session. Acceptable for
  an action you start and walk away from, and the honest price of self-sufficiency.
- **Two processing paths exist**, with the complexity that implies: two failure modes, two
  sets of config, and a routing rule. Accepted as the cost of not forcing a deployment on
  someone who does not want one.
- **Preemption survives, and only for the local path.** Local transcription yields to
  recording because it holds CPU on the capturing machine; a service job does not. One
  rule — *whatever holds CPU on the Pi yields to capture* — with the route determining
  whether anything does.
- **Diarization is a capability, not a guarantee.** A user without a host does not get
  speaker labels. Stating that plainly is better than a cloud dependency that quietly
  makes it work.
- **No API keys exist anywhere in the system.** FR-26 stays retired.
- **The service must tolerate being absent**, and the device must treat that as normal
  configuration rather than an error state.

## Alternatives

- **Service required for all processing** — the earlier draft. Rejected: it makes a
  standalone recorder impossible and turns a plug-in appliance into a two-component
  system. Everything it removed from the Pi was worth less than what it took away from
  the product.
- **Local processing only, no service** — maximum simplicity. Rejected: it forfeits
  diarization entirely and leaves transcription at 20–37 minutes per session for users
  who have hardware that could do it in a fraction of the time.
- **Local transcription plus a cloud API for diarization** — where this started. Rejected:
  the key, the cost, the per-request limits, and an internet dependency for a product
  whose point is that your conversations stay yours.
