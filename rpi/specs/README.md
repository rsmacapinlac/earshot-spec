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
| [transcription.md](transcription.md) | Queue, scheduling, process, transcript format (FR-14–FR-18) |
| [usb-offload.md](usb-offload.md) | FAT32 USB-stick offload on Pi 4B (FR-11) |
| [install-service.md](install-service.md) | Installer steps and the systemd unit contract (FR-8, FR-10) |

## Conventions
- **FR-n** identifiers are stable requirement IDs so the implementation and the
  companion `earshot-tui` can trace behavior back to a spec. IDs are never reused;
  a retired requirement (e.g. FR-5, the removed speaker output) leaves a gap.
- *Implementation note* callouts flag intended implementation specifics (paths,
  unit fields, backends) that sit below the normative contract.
