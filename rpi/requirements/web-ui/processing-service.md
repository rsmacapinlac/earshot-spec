# Configure a processing service

Set, update, and clear the processing service URL, and show live connection status.

**Optional by design.** Unset is a normal state and must be presented as such, never as an
error: the device transcribes locally without it
([optional processing service](../../adr/optional-processing-service.md)). Configuring one makes
transcription much faster and is the only way to enable
[diarization](diarize.md).

## Status

Three states, each meaning something different to the user:

| State | Meaning |
|---|---|
| **Not set** | Transcription runs on this device. Diarization unavailable. |
| **Unreachable** | A URL is configured but the service is not responding. Transcription falls back to running locally. |
| **Connected** | Shows the capabilities the service actually offers. |

**Capabilities come from the service, not from the URL being set.** The device discovers
them from the service itself — reachability for transcription, and probing its
`/openapi.json` for the `diarize` parameter
([service API](../../reference/processing-service.md#verifying)). A
deployment with transcription but no diarization models must offer one action, not two that
fail differently.

- Applies without a restart, unlike other settings.
- Persists to `[processing].service_url`
  ([configuration.md](../../specs/configuration.md#processing)).
- An unreachable service is a *connection* problem, reported as such — it does not count
  against any individual session's retry budget
  ([processing.md](../../specs/processing.md#failure)).

Deploying the service itself is out of scope for this UI; see
[service deployment](../../reference/processing-service.md).
