# Firmware Specs

Specs are implementation contracts: exact behavior, file formats, state transitions, hardware-facing numbers, and interfaces the firmware should satisfy.

Product/user requirements live in `../requirements/`. Decision rationale lives in `../adr/`.

## Specs

- `state-machine.md` — states, transitions, button semantics, and condition flows.
- `storage.md` — SD-card recording layout, metadata, labels, scanning, commit, and delete behavior.
- `recording-playback.md` — audio format, capture, WAV, and playback rules.
- `power-sleep.md` — battery gauge, power rails/latch, and sleep behavior.
- `boot-configuration.md` — `/earshot.cfg` scaffold behavior for future settings.
