# Earshot (RPi) — Specs Index

Normative behavior for the Raspberry Pi Earshot: exact thresholds, file formats,
state transitions, and contracts. These specs reflect the **as-built v0.2.2**
code on `pi-earshot-pi4`; where the implementation's own docs disagreed, the code
wins.

Product needs and scope: [`../requirements/`](../requirements/README.md).
Rationale: [`../adr/`](../adr/README.md).

## Documents

| File | Covers |
|---|---|
| [configuration.md](configuration.md) | `config.toml` schema — every key, type, default (as-built) |
| [state-machine.md](state-machine.md) | States, LED colour/pattern table, button semantics (FR-1–FR-5) |
| [recording.md](recording.md) | Capture spec, chunking, concatenation into a single WAV (FR-2a, FR-3, FR-6) |
| [storage.md](storage.md) | Session layout, filesystem-as-state, disk management, crash recovery (FR-7) |
| [transcription.md](transcription.md) | Queue, scheduling, process, transcript format (FR-14–FR-18) |
| [usb-offload.md](usb-offload.md) | FAT32 USB-stick offload on Pi 4B (FR-11) |
| [install-service.md](install-service.md) | Installer steps and the systemd unit contract (FR-8, FR-10) |

## Conventions
- **FR-n** identifiers are carried over from the implementation's requirements so
  they remain traceable to the code and to `earshot-tui`.
- *As-built* callouts mark behavior verified against the running code/device that
  differs from the implementation's own written docs.
