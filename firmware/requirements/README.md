# earshot — Product Requirements

earshot is custom firmware for a pocket e-paper **voice-note recorder** built on
the Waveshare ESP32-S3 1.54" e-Paper board (`S3_ePaper_1_54`, ESP32-S3-PICO-1).

This directory captures product and quality requirements: what the device must do
and what constraints matter to users. Exact implementation contracts live in
`../specs/`; rationale for major choices lives in `../adr/`.

## v1 scope

A standalone offline recorder that lays the foundation for future BLE sync.

v1 must let the user:

- record mono voice notes to an SD card;
- identify recordings by stable recording ID and optional spoken label, without
  needing date/time setup;
- browse, play, and delete recordings;
- receive clear no-SD, storage-full, sleep, charging, and low-battery feedback;

Explicitly out of v1, but designed-for:

- BLE file sync to a future iPhone/Android companion app. v1 ships a transfer
  seam, not the feature.

## Requirement documents

- `non-functional.md` — battery, responsiveness, durability, storage, and other
  cross-cutting quality requirements.
- `open-ux-questions.md` — deferred interaction/UX decisions.
- `open-technical-decisions.md` — deferred engineering decisions.

## Related implementation specs

- `../specs/README.md` — index of precise firmware behavior/contracts.

## Related references and decisions

- `../reference/` — board/panel facts and hardware references.
- `../adr/README.md` — architecture decision records and rationale.
