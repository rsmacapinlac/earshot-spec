# AGENTS.md

You are working in the **earshot spec repository**. Treat this repo as the source of product requirements, firmware specs, ADRs, and hardware references for the earshot project.

## Documentation roles

Keep these boundaries clear:

- **Requirements** describe product/user needs and cross-cutting qualities.
- **Specs** define exact firmware behavior, file formats, thresholds, state transitions, and contracts.
- **ADRs** explain important architectural choices and alternatives. Do not duplicate full specs in ADRs.
- **Reference** documents hardware facts or non-normative bring-up/history notes.
- **Experiments** collect data to drive a design decision. Each one names the open
  decision it supports (a TD, or a pending ADR), states a hypothesis and success
  criteria, runs a procedure that gathers measurements, and concludes with a
  recommendation that updates the design. They are non-normative and time-bound —
  the evidence trail behind a decision. An experiment that does not lead to a
  decision does not belong here.

Docs are organized by product track: `esp32/` (the ESP32-S3 e-paper firmware) and
`rpi/` (the Raspberry Pi recorder application). Each track carries the same
subdirectories. Within the relevant track: when a doc contains exact implementation
behavior, prefer placing it in that track's `specs/`. When a doc explains why a
major approach was chosen, use its `adr/`. When a doc validates or measures behavior
on hardware to resolve an open question, use its `experiments/` (named
`NNNN-slug.md`, sequential like ADRs). Once an experiment resolves something, fold
the result into the relevant spec/ADR and update the related TD, leaving the
experiment as the evidence record.

## Editing guidelines

- Keep links valid after moving/deleting files.
- After deleting or renaming docs, search the repo for stale references.
- Keep specs concise but testable.
- Keep ADRs focused on explaining the decision, alternatives that were explored, and consequences.
- Keep reference docs factual and non-normative.
- Keep experiments hypothesis-driven and testable: name the design decision each
  one supports, state success criteria up front, and collect data against them;
  when an experiment closes the decision, update the spec/ADR/TD it informed.

Be concise, direct, and collaborative. The user is actively shaping product and firmware architecture; help clarify tradeoffs, then update the docs cleanly.

## ESP32 Firmware Section

The `esp32/` firmware documentation is **v1.7**. See `esp32/CHANGELOG.md` for the
version history.

The firmware implementation lives in the separate GitHub repository:

```text
https://github.com/rsmacapinlac/earshot-firmware
```

Only inspect a local checkout of that repository when explicitly asked to compare these specs to the current firmware implementation.

## RPi Section

The `rpi/` documentation is a **separate product** from the `esp32/` firmware
track: a Raspberry Pi application (Pi 4B + Seeed ReSpeaker 2-Mic HAT) that records
conversations locally and offloads audio via USB. It follows the same
documentation roles and structure as the `esp32/` track
(`requirements/`, `adr/`, `specs/`, `reference/`, `experiments/`).

This RPi documentation is **v1.0**. See `rpi/CHANGELOG.md` for the version history.

The implementation lives in the separate GitHub repository:

```text
https://github.com/rsmacapinlac/earshot
```

Only inspect a local checkout of that repository when explicitly asked to compare
these specs to the current implementation.


