# Architecture Decision Records (RPi)

Significant architectural and technical decisions for the Raspberry Pi Earshot,
with the context and reasoning behind them. Imported from the implementation
repo's own ADRs and reconciled against the as-built v0.2.2 code; *As-built* notes
flag where the original decision text no longer matches the running system.

## Format
- **Status** — Proposed, Accepted, Deprecated, or Superseded
- **Context** — the problem that required a decision
- **Decision** — what was decided
- **Consequences** — trade-offs and implications

## Index

| ADR | Title | Status |
|---|---|---|
| [0001](0001-audio-storage-format.md) | Audio stored as a single WAV file | Accepted |
| [0002](0002-python-venv-over-docker.md) | Python venv over Docker | Accepted |
| [0003](0003-hardware-abstraction-layer.md) | Hardware abstraction layer | Accepted |
| [0004](0004-systemd-for-service-management.md) | systemd for service management | Accepted |
| [0006](0006-filesystem-as-state.md) | Filesystem as state, no SQLite | Accepted |
| [0007](0007-chunked-recording.md) | Chunked recording (15-minute default) | Accepted |
