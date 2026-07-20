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
| [recording.md](recording.md) | Capture spec, chunking, concatenation into a single WAV (FR-2a, FR-3, FR-6) |
| [storage.md](storage.md) | Session layout, filesystem-as-state, disk management, crash recovery (FR-7) |
| [transcription.md](transcription.md) | Queue, web-initiated trigger, process, transcript format, diarization (FR-14–FR-18, FR-25) |
| [install-service.md](install-service.md) | Installer steps and the systemd unit contract (FR-8, FR-10) |

## Conventions
- **FR-n** identifiers are stable requirement IDs so the implementation and the
  companion `earshot-tui` can trace behavior back to a spec. IDs are never reused;
  retired or reserved requirements leave gaps.

Retired/reserved IDs:
- `FR-5` — removed speaker/audio-output behavior.
- `FR-11` — retired.
- `FR-12` / `FR-13` — reserved by earlier drafts and intentionally unused in v1.0.

`FR-19`–`FR-27` (web UI, recording control, on-demand transcription, diarization,
speaker enrollment) are defined in
[`../requirements/web-ui.md`](../requirements/web-ui.md); a dedicated `web-ui.md` spec
will follow once TD-7 is validated.

Implementation notes:
- *Implementation note* callouts flag intended implementation specifics (paths,
  unit fields, backends) that sit below the normative contract.
