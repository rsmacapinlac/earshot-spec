# Changelog

All notable changes to the earshot **Raspberry Pi documentation** (specs,
requirements, ADRs, reference, experiments). The documentation set is versioned as
a whole; the current version is recorded in `../AGENTS.md`. Dates are ISO-8601
(YYYY-MM-DD).

## [Unreleased]

_Nothing yet._

## [1.0] — 2026-07-19

### Added
- **Initial RPi documentation set**, reverse-engineered from the running
  implementation on host `pi-earshot-pi4` (app **v0.2.2**) and reconciled against
  the actual code and device state. Structured to mirror the ESP `esp32/`
  track: `requirements/`, `adr/`, `specs/`, `reference/`, `experiments/`.
- **Requirements** (`requirements/`): product scope, supported hardware (Pi 4B +
  ReSpeaker 2-Mic HAT), non-functional targets, connectivity, and the
  `open-technical-decisions.md` (TD-n) / `open-ux-questions.md` (UX-n) registries.
- **ADRs** (`adr/0001…0007, 0010`): imported from the implementation's own ADRs,
  each carrying an *As-built* note where the original decision text had drifted
  from the v0.2.2 code.
- **Specs** (`specs/`): normative configuration schema, state machine + LED table,
  recording/encoding, storage/filesystem-state, transcription, USB offload, and
  the installer + systemd service contract. `FR-n` identifiers preserved for
  traceability to the code and `earshot-tui`.
- **Reference** (`reference/respeaker-2mic-hat.md`): ReSpeaker 2-Mic HAT hardware
  facts and the observed WM8960 mixer/boot configuration.
- **Experiments** (`experiments/`): scaffolding (README + TEMPLATE) and
  Experiment 0001 (capture gain / ALC tuning) supporting TD-1.

### Fixed (as-built corrections vs. the implementation's own docs)
- Captured/stored audio is **stereo** (both mics) — see TD-2.
- Transcription engine is **faster-whisper**, not whisper.cpp.
- Documented the **real `config.toml` schema** (`[audio].alsa_pcm`,
  `[recording].shutdown_hold_seconds`, `[storage].data_dir`), not the older
  `[encoding]`/`[shutdown]` schema in the source docs.

### Audio format
- Audio is stored as a single **`session.wav`** (chunks concatenated at session
  end); `session.wav` is the offloaded and transcribed artifact. See
  [ADR-0001](adr/0001-audio-storage-format.md).

### Notes
- Scope is **Pi 4B + ReSpeaker + USB-A offload** (the as-built configuration). The
  Pi Zero 2W + USB-gadget-mode path is design intent only and is not specified
  normatively — tracked as TD-3.
