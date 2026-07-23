# Core functionality over the API

## Requirement

**Every core operation of the device is reachable through the HTTP API.** Recording
control, listing and playing recordings, download, deletion, transcription, diarization,
speaker and session naming, and device status are all API operations. No core operation
requires reading files directly, querying the database, or opening a shell.

This is what makes the web UI a *client* rather than *the* interface, and what lets a
second client — a script, a future companion app — do anything the UI can
([the HTTP API is the interface](../../adr/http-api-is-the-interface.md)).

## Operating vs administering

The requirement covers **operating** the device. It does not extend to setup:

| Task | How | Why |
|---|---|---|
| Operate — record, browse, transcribe, name, status | **API** | The device's job; done constantly. |
| Configure — `config.toml` | **SSH** | Administration, done rarely, by whoever installed it ([configuration.md](../../specs/configuration.md)). |
| Install, network setup | **SSH** | No service to call yet, and you cannot configure the network over the network ([connectivity.md](../connectivity.md)). |

Editing configuration over SSH is not a gap in the API — configuration is deliberately not
a core operation. The one setting the UI exposes, the processing service URL, is there
because *connecting to a service* is operational, not because config is drifting into the
API.

## Not in scope

- Authentication. Trusted-LAN, no login in v1
  ([serve over the LAN](../web-ui/serve-over-lan.md)). This is about the API being
  complete, not about who may call it.
