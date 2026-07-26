# Diarize (speaker labels)

Diarization adds *who spoke* to *what was said*. It is an **option on
[transcribe](transcribe.md)**, not a separate action: check the **Diarize** box when
transcribing to get a speaker-labelled transcript, or leave it off for a plain one. You
never have to diarize.

**Requires a [processing service](processing-service.md).** There is no local path and no
cloud path: the models need compute a Pi 4B does not have, and reaching for a third-party
API to work around that would trade the product's independence for a feature
([optional processing service](../../adr/optional-processing-service.md)).

The Diarize option is **shown only when a service is configured and reports the
capability** — a deployment with transcription but no diarization models simply doesn't
offer the checkbox, rather than presenting an action that fails.

## Behaviour

- **The diarized result replaces the transcript.** The service transcribes and labels in
  one job, so its output *is* the transcript; there is never a second, divergent transcript
  of the same audio. Diarizing does not require a prior transcript.
- **Speakers come back generic** — `Speaker 1`, `Speaker 2`, … — and are named afterward
  ([naming speakers](name-speakers.md)). An optional **speaker-count hint** may be supplied
  when the number of talkers is known.
- **Length is not a special case.** The service clusters over the whole recording in one
  pass, so a voice keeps one label from beginning to end however long the session is.
- **On failure the existing transcript is left intact.** The overwrite happens only on
  success.
- **Reverting is always possible**, even after the service is gone: a local re-transcribe
  removes the speaker labels.
- **In bulk:** "transcribe all" with Diarize checked processes every not-yet-diarized
  session at once ([transcribe](transcribe.md)).

Quality is bounded by the mono, closely-spaced-mic capture. It is not gated by any
decision — with the models on your own hardware each attempt costs nothing, so the user
judges it on their own audio.

Contract: [`specs/processing.md`](../../specs/processing.md#diarization).
