# Earshot (RPi) — Specs Index

Normative behavior for the Raspberry Pi Earshot: exact thresholds, file formats,
state transitions, and contracts. These specs define the **target behavior** the
implementation must meet; they are the authoritative source when other docs
disagree.

Product needs and scope: [`../requirements/`](../requirements/README.md).
Rationale: [`../adr/`](../adr/README.md).

## Documents

| File | Covers |
|---|---|
| [configuration.md](configuration.md) | `config.toml` schema — every key, type, default |
| [state-machine.md](state-machine.md) | States, LED colour/pattern table, button semantics (FR-1–FR-4) |
| [recording.md](recording.md) | Capture spec, chunking, encoding into a single m4a (FR-2a, FR-3, FR-6) |
| [storage.md](storage.md) | File layout, the state database, reconciliation, disk management, crash recovery (FR-7) |
| [processing.md](processing.md) | Local and service transcription routes, queue, preemption, transcript format, diarization (FR-14–FR-18) |
| [install-service.md](install-service.md) | Installer steps and the systemd unit contract (FR-8, FR-10) |
| [api.md](api.md) | The HTTP API the web UI is a client of — sessions, recording control, jobs, device status |

## Conventions
- **FR-n** identifiers are stable requirement IDs so the implementation can trace behavior
  back to a spec. IDs are never reused; retired or reserved requirements leave gaps.

Retired/reserved IDs:
- `FR-5` — removed speaker/audio-output behavior.
- `FR-11` — retired.
- `FR-12` / `FR-13` — reserved by earlier drafts and intentionally unused in v1.
- `FR-19`–`FR-30` — retired as identifiers. Those were the web UI capabilities, now named
  rather than numbered ([`../requirements/web-ui/`](../requirements/web-ui/README.md)).

Web UI capabilities — recording control, on-demand processing, diarization, speaker and
session naming, device status, service configuration — are defined in
[`../requirements/web-ui/`](../requirements/web-ui/README.md) and referred to **by name
rather than by number**; the HTTP surface those capabilities are called through is
[api.md](api.md).

Implementation notes:
- *Implementation note* callouts flag intended implementation specifics (paths,
  unit fields, backends) that sit below the normative contract.
