# Web UI

The Raspberry Pi is headless with a single LED as its only local feedback channel. The web
UI — served by the application over the LAN, reached at the Pi's IP — is the surface for
everything the button cannot express. Given the button has exactly two gestures, and both
are taken by record/stop and shutdown-hold, that is nearly everything.

Each capability has its own file, named for what it is; they are referred to by name
rather than by number.

## Capabilities

| Capability | In short |
|---|---|
| [Serve over the LAN](serve-over-lan.md) | The UI itself — bound to the Pi's IP, trusted LAN, no auth in v1 |
| [List sessions](list-sessions.md) | Every session with derived status, identity, duration, size |
| [Play and download](play-and-download.md) | Listen in the browser; download the audio |
| [Delete a session](delete-session.md) | Remove a session and free the disk, with confirmation |
| [Recording control](recording-control.md) | Start and stop, shared with the hardware button |
| [Transcribe](transcribe.md) | On demand, locally or on a service — always available |
| [Diarize](diarize.md) | Label who spoke — requires a processing service |
| [Name the speakers](name-speakers.md) | Put real names on `Speaker N`, per session, after the fact |
| [Device status](device-status.md) | Live mirror of what the LED is showing |
| [Name a session](name-session.md) | An optional human label on top of the session ID |
| [Set a session date and time](set-session-datetime.md) | An optional user-asserted date/time, since the device clock can't be trusted |
| [Configure a processing service](processing-service.md) | Optional URL, with live connection and capability status |

## Settled decisions

- **Parity for recording, extension for processing.** Record, stop, and *status* are the
  same capability on two surfaces — button plus LED, and the web UI. Transcription and
  diarization exist **only** on the web surface, and should not acquire button gestures.
  As extensions they must never degrade recording, the base function.
- **The device stands alone; a service is an upgrade.** Transcription always works —
  locally by default, or on a [processing service](processing-service.md) if one is
  configured. A fresh Pi with no configuration transcribes. Removing a service falls back;
  nothing is lost ([optional processing service](../../adr/optional-processing-service.md)).
- **Diarization is the one capability that needs a service.** Without one the action is
  simply not offered — presented as a capability this deployment lacks, not as a failure.
- **The UI offers only what is actually available**, based on what a service reports
  rather than on a URL merely being set.
- **One transcript per session; diarization replaces it.** Every session has a single
  `transcript.md`, rendered by the device. A diarize job overwrites it with the
  speaker-labelled version. No separate diarized file is kept.
- **Speakers are named after the fact, per session** — no enrollment, no cross-session
  registry, no identity carried between recordings.
- **Sessions are identified by name, not by date.** Identity is the allocated session ID;
  the name is a label on top of it, and the capture date is a scanning convenience only.
- **No API keys and no third parties.** There is no cloud path for anything.
- **The UI is a client of the device's API, not a privileged path**
  ([the HTTP API is the interface](../../adr/http-api-is-the-interface.md)). Every core
  operation is reachable over the API
  ([core functionality over the API](../non-functional/core-functionality-over-api.md)), so
  anything the UI does, a script or companion app can too.
- **The UI operates the device; it does not configure it.** Recording, browsing,
  transcribing, naming, status — all here. Editing `config.toml` is an SSH operation
  ([configuration.md](../../specs/configuration.md)); the one exception is the processing
  service URL, which is an operational connection, not general config.

## Out of scope (v1)

- Authentication or user accounts.
- Speaker enrollment, and any cross-session speaker registry.
- Summarization — a designed-for future action, not built.
- Editing `config.toml`, beyond the processing service URL. Configuration is done over
  SSH — the intended workflow, since the device was installed that way and settings change
  rarely ([configuration.md](../../specs/configuration.md)).
- Deploying or managing a processing service. The UI points at one if you have it;
  standing it up is a separate operation
  ([service deployment](../../reference/processing-service.md)).

## Where the behavior is specified

- [`specs/processing.md`](../../specs/processing.md) — local and service routes, queue,
  preemption, transcript format, diarization.
- [`specs/state-machine.md`](../../specs/state-machine.md) — web-initiated start/stop and
  the Processing state.
- [`specs/api.md`](../../specs/api.md) — the HTTP contract these capabilities are called
  through.
- [`specs/configuration.md`](../../specs/configuration.md) — the config schema.
- [`specs/storage.md`](../../specs/storage.md) — the file layout, the state database, and
  how the two reconcile.
- [`connectivity.md`](../connectivity.md) and
  [`non-functional/`](../non-functional/README.md) — LAN-only; no internet required.
- [`reference/processing-service.md`](../../reference/processing-service.md) — the optional
  processing service this UI can drive.

## Still to capture

Present in the design prototype but not yet written up as requirements: exporting
`transcript.md` (only audio is downloadable today), live state reflection across surfaces
so the UI follows a physical button press without a reload, and feedback for events the
user did not initiate — such as a local transcription being cancelled because someone
walked up and pressed record.
