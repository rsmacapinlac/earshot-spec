# The HTTP API Is the Interface

**Status:** Accepted

## Context

The device is headless; the web UI over the LAN is how you operate it. That leaves one
question open: is there a real API behind the UI, or does the UI reach into the database
and files directly?
[Core functionality over the API](../requirements/non-functional/core-functionality-over-api.md)
requires the former.

## Decision

**The application exposes one HTTP API, and the web UI is a client of it** — no privileged
internal path. Everything the browser does, it does through endpoints any client could
call. The API, not the on-disk layout, is the device's public contract for operation. It
is specified in [specs/api.md](../specs/api.md).

Configuration is out of scope for the API by design: editing `config.toml` is an SSH
operation ([configuration.md](../specs/configuration.md)), because it is administration,
not a core operation. The API is how you *operate* the device.

## Consequences

- **The API is the thing to get right, not the UI.** A script or a future companion app is
  a first-class client, because the UI has no capability it does not.
- **The on-disk layout is an implementation detail, not a public contract.** Session
  directories, `status.json`, and the database can change shape without breaking clients,
  because clients use the API — not `scp`, not file reads. Recordings and transcripts are
  fetched over HTTP.
- **[`specs/api.md`](../specs/api.md) is the real HTTP contract**, distinct from the
  capability descriptions in
  [`requirements/web-ui/`](../requirements/web-ui/README.md), which say *what* the UI offers
  rather than *how* it is called.

## Alternatives

- **Web UI as a monolith** reading the database and filesystem directly, with a thin or
  absent public API. Rejected: it makes the UI and the API two different things, so a
  second client is second-class and "the API is complete" is never actually true.
- **Config as an API resource too**, so the device needs no shell at all. Rejected:
  editing `config.toml` over SSH is fine for a device installed over SSH and configured
  rarely, and a live-config API is real machinery — settings endpoints, validation,
  self-restart for the settings that need it — for little gain
  ([configuration.md](../specs/configuration.md)).
