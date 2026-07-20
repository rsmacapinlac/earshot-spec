# Changelog

All notable changes to the earshot **Raspberry Pi documentation** (specs,
requirements, ADRs, reference, experiments). The documentation set is versioned as
a whole; the current version is recorded in `../AGENTS.md`. Dates are ISO-8601
(YYYY-MM-DD).

## [Unreleased]

### Added
- **Capture front-end spec** (`specs/recording.md`): the WM8960 is configured for
  ALC using Wolfson's speech preset (target −12 dBFS, fast 24 ms attack / 384 ms
  decay, noise gate + HPF on, Max Gain capped at 5 provisionally, on the captured
  left channel), with the `amixer` values and the requirement to persist them to
  `/etc/voicecard/wm8960_asound.state`.

### Changed
- **TD-1 resolved and removed.** The capture-gain question (fixed PGA vs. ALC) is
  decided in favour of ALC and folded into `specs/recording.md`; the
  `reference/` front-end section now points to the spec. `ALC Max Gain` ships as a
  starting value (5) to confirm on hardware during bring-up.
- **TD-2 resolved and removed.** Capture is now **mono** (the left mic), not
  stereo: faster-whisper downmixes to mono anyway, the closely-spaced mics carry
  no usable stereo image, and mono halves the WAV size. A single channel is taken
  (not an L+R average) to avoid comb filtering on off-axis talkers. `config.toml`
  default `audio.channels` becomes `1`; `recording.md`/`configuration.md`/
  `storage.md` updated.
- **TD-3 resolved and removed — Pi 4B is the minimum; Pi Zero 2W is out of scope.**
  Dropped the Pi Zero 2W + USB-gadget-mode design-intent content from
  `hardware.md`, `usb-offload.md`, `transcription.md`, and the systemd capability
  rationale (`CAP_SYS_MODULE`/`CAP_SYS_ADMIN` no longer needed).
- **TD-4 resolved and removed — default transcription model is now `base.en`**
  (was `tiny.en`), for better accuracy on the Pi 4B; `tiny.en` stays available as
  the lighter alternative. `configuration.md`/`transcription.md` updated. The
  technical-decisions registry is now empty.
- **UX-1/UX-2/UX-3 resolved and removed** — all kept the v1 behavior already in
  the specs: LED colour overload accepted (single LED, disambiguated by pulse
  speed); audio feedback is out of scope; and the single-button gestures are
  unchanged (3 s idle hold = shutdown, no confirmation — low-stakes since captures
  commit first). The UX registry is now empty.
- **Removed all speaker / audio-output content** — the hardware isn't present.
  Dropped backlog B-A1/B-A2 (audio cues + volume), FR-5, the `AudioOutputInterface`
  mention (hardware abstraction layer ADR), and the Speaker rows from `hardware.md`
  and `reference/`.
  The LED is the sole feedback channel.
- **Backlog triaged.** Dropped B-T2 (installer model prompt) and B-I2 (Pi 5
  support). **B-I1 (Web UI / dashboard) promoted to the next release.**

## [1.0] — 2026-07-19

### Added
- **Initial RPi documentation set** — the target specification for the Raspberry
  Pi Earshot application (not yet built). Structured to mirror the ESP `esp32/`
  track: `requirements/`, `adr/`, `specs/`, `reference/`, `experiments/`.
- **Requirements** (`requirements/`): product scope, supported hardware (Pi 4B +
  ReSpeaker 2-Mic HAT), non-functional targets, connectivity, and the
  `open-technical-decisions.md` (TD-n) / `open-ux-questions.md` (UX-n) registries.
- **ADRs** (`adr/`): audio storage format, Python venv over Docker, hardware
  abstraction layer, systemd for service management, filesystem-as-state, and
  chunked recording — each with an *Implementation note* flagging intended
  implementation specifics.
- **Specs** (`specs/`): normative configuration schema, state machine + LED table,
  recording, storage/filesystem-state, transcription, USB offload, and the
  installer + systemd service contract. `FR-n` identifiers give the implementation
  and `earshot-tui` stable requirement IDs to trace behavior to.
- **Reference** (`reference/respeaker-2mic-hat.md`): ReSpeaker 2-Mic HAT hardware
  facts and the expected WM8960 mixer/boot configuration.
- **Experiments** (`experiments/`): scaffolding (README + TEMPLATE) for future
  hardware validation.

### Decisions baked into the spec
- Capture is **mono** (the left mic), 16 kHz / 16-bit PCM.
- Transcription engine is **faster-whisper**.
- The `config.toml` schema is `[audio].alsa_pcm`,
  `[recording].shutdown_hold_seconds`, `[storage].data_dir` (not an
  `[encoding]`/`[shutdown]` schema).

### Audio format
- Audio is stored as a single **`session.wav`** (chunks concatenated at session
  end); `session.wav` is the offloaded and transcribed artifact. See the
  [audio storage format ADR](adr/0001-audio-storage-format.md).

### Notes
- Scope is **Pi 4B + ReSpeaker + USB-A offload**.
