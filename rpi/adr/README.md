# Architecture Decision Records (RPi)

Significant architectural and technical decisions for the Raspberry Pi Earshot, with the
context and reasoning behind them — and, importantly, the alternatives that were rejected
and why.

Each record gets its own file, named for the decision. They are referred to **by name, not
by number**, so nothing has to be looked up in an index to be understood.

## Format
- **Status** — Accepted, Deprecated, or Superseded, with the date of any re-decision
- **Context** — the problem that required a decision
- **Decision** — what was decided
- **Consequences** — trade-offs and implications, including the costs
- **Alternatives** — what else was considered, and why it was not chosen

A decision that changes is **re-decided in place** rather than superseded by a new record,
with a *History* section explaining what changed. Nothing has been built against these
yet, so a superseding chain would only make a reader visit a void record before reaching
the live one.

## Decisions

| Decision | In short |
|---|---|
| [Audio storage format](audio-storage-format.md) | Sessions stored as a single `session.m4a`, AAC-LC 32 kbps |
| [Chunked recording](chunked-recording.md) | Audio captured in 15-minute WAV chunks so a crash costs at most one |
| [Session identity](session-identity.md) | Identity is a database-allocated integer, never a timestamp, never reused |
| [State storage](state-storage.md) | SQLite for state, files for artifacts |
| [Job execution](job-execution.md) | An in-process worker over a table — no broker, no task-queue framework |
| [The HTTP API is the interface](http-api-is-the-interface.md) | One API; the web UI is a client of it, and the on-disk layout is not a public contract |
| [Optional processing service](optional-processing-service.md) | The device stands alone; a processing service is an upgrade, never a dependency |
| [Hardware abstraction layer](hardware-abstraction-layer.md) | Button, LED and capture behind interfaces, with a stub for development |
| [systemd for service management](systemd-for-service-management.md) | Runs as a systemd unit, started on boot by the installer |
| [Python venv over Docker](python-venv-over-docker.md) | A venv on the device — direct GPIO, SPI and ALSA access without privileged containers |

Product requirements live in [`../requirements/`](../requirements/README.md); precise
implementation specs live in [`../specs/`](../specs/README.md).
