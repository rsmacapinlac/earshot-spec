# Diarize a session

Label who spoke, in addition to what was said.

**Requires a [processing service](processing-service.md).** There is no local path and no
cloud path: the models need compute a Pi 4B does not have, and reaching for a third-party
API to work around that would trade the product's independence for a feature
([optional processing service](../../adr/optional-processing-service.md)).

Offered only when a service is configured **and reports the capability** — a deployment
with transcription but no diarization models shows one action, not two that fail
differently.

## Behaviour

- **The diarized result replaces the transcript.** The service transcribes and labels in
  one job, so its output *is* the transcript; there is never a second, divergent
  transcript of the same audio. Diarizing does not require a prior transcript.
- **Speakers come back generic** — `Speaker 1`, `Speaker 2`, … — and are named afterward
  ([naming speakers](name-speakers.md)).
- **Length is not a special case.** The service clusters over the whole recording in one
  pass, so a voice keeps one label from beginning to end however long the session is.
  There is no split and no session length at which behaviour changes.
- **On failure the existing transcript is left intact.** The overwrite happens only on
  success.
- **Reverting is always possible**, even after the service is gone: a local re-transcribe
  removes the speaker labels.

Quality is bounded by the mono, closely-spaced-mic capture. It is not gated by any
decision — with the models on your own hardware each attempt costs nothing, so the user
judges it on their own audio.

Contract: [`specs/processing.md`](../../specs/processing.md#diarization).
