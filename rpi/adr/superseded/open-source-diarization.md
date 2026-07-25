# Open-Source Diarization (pyannote), Not a Cloud API

**Status:** Superseded (2026-07-25) by [Adopt an off-the-shelf processing service](../off-the-shelf-processing-service.md).

> **Superseded.** This decision was made for a bespoke earshot-built service. earshot no
> longer builds one — it adopts an off-the-shelf service, and "runs locally / no cloud
> diarization / consistent across a whole recording" is now a **selection criterion** in
> [Adopt an off-the-shelf processing service](../off-the-shelf-processing-service.md),
> validated for the adopted WhisperX service in
> [experiment 0002](../../experiments/0002-whisper-asr-webservice.md). Kept for history.

## Context

Diarization previously ran against OpenAI's `gpt-4o-transcribe-diarize`. That was not a
quality judgement — it was a hardware one. Nothing of the required quality would run on a
Pi 4B, so the work went to the only host available, which was somebody else's.

The import cost:

- **25 minutes per request.** Longer recordings had to be split.
- **Speaker labels uncorrelated across requests.** A split recording returned
  `Speaker 1` in each part with no relationship between them, so the same person appeared
  as several speakers and the user had to name each one.
- An API key to store, protect, and keep out of version control.
- Per-session cost.
- The product's only dependency on the internet.

Once processing moved to a service on a host chosen for the job
([separate processing service](separate-processing-service.md)), the hardware constraint that forced
all of this disappeared.

## Decision

Diarization runs **in the service, on open-source models** — pyannote
`speaker-diarization-3.1` producing speaker turns, assigned onto faster-whisper segments
by overlap ([processing.md](../../specs/processing.md#diarization)).

No cloud provider is called. No API key exists in the system.

## Consequences

- **Speaker labels are consistent across an entire recording**, of any length. pyannote
  clusters embeddings over the whole audio rather than per request, which is precisely
  the property the chunked cloud API could not offer.
- **TD-7 stops existing.** No 25-minute cap, no splitting, no cross-request stitching, no
  duplicate `Speaker N` entries for one person. A rule the device had to explain to users
  is simply gone.
- **No key, no cost, no internet.** Key management is retired from the device, and with it
  the last thing that made earshot depend on a third party.
- **Cost: weights are large and gated.** pyannote's pretrained models need a HuggingFace
  token and terms acceptance, once, at setup. If they are absent the service still starts
  and serves transcription, reporting `diarize: false` with the reason (SNFR-4) — but a
  first-run step now exists that can go wrong.
- **Cost: quality is ours to own.** Previously a vendor's model was a fixed input; now
  diarization accuracy is a property of the models we ship and can tune. Whether pyannote
  on mono, closely-spaced-mic audio is good enough remains untested — but it is testable
  on our own hardware, repeatedly, without paying per attempt.
- **Diarization needs real compute.** This is what makes the host requirements in
  [deployment reference](../../reference/processing-service.md#host-requirements) non-trivial, and it is why a
  Pi is explicitly not a supported host.

## Alternatives

- **Keep OpenAI, called from the service** — smallest change; the service would just
  relocate the call. Rejected: it relocates the problem too. Every constraint above is a
  property of the provider, and moving where the request originates fixes none of them.
- **Support both backends, configurable** — flexibility. Rejected for v1: two result
  shapes, two failure modes, two sets of documentation, for a choice that only exists
  because of a hardware constraint we have now removed. Reconsider only if pyannote
  proves inadequate in practice.
- **WhisperX** rather than faster-whisper plus pyannote directly — packages the same
  combination with forced alignment for word-level timestamps. Not rejected on merit; it
  remains a reasonable substitution if word-level timing is wanted later. The pipeline is
  specified in terms of stages, so swapping the implementation does not change the API.
