# Changelog

All notable changes to the earshot **Raspberry Pi documentation** (specs,
requirements, ADRs, reference, experiments). The documentation set is versioned as
a whole; the current version is recorded in `../AGENTS.md`. Dates are ISO-8601
(YYYY-MM-DD).

## [Unreleased]

### Added
- **Web UI requirement** (`requirements/web-ui.md`): a LAN-served web UI (FR-19–FR-27)
  to browse / listen / delete recordings, start/stop recording, initiate transcription,
  and — with an OpenAI key — diarize sessions via `gpt-4o-transcribe-diarize` and enroll
  named speakers. Trusted-LAN / no-auth in v1; the OpenAI key lives in `config.toml`
  (`[diarization].api_key`) and is settable from the web UI.
- **Experiment 0001** (`experiments/0001-openai-diarization-mono-and-chunking.md`):
  validates OpenAI diarization quality on mono ReSpeaker audio and cross-part speaker
  stitching (supports TD-7).
- **Capture front-end spec** (`specs/recording.md`): the WM8960 is configured for
  ALC using Wolfson's speech preset (target −12 dBFS, fast 24 ms attack / 384 ms
  decay, noise gate + HPF on, Max Gain capped at 5 provisionally, on the captured
  left channel), with the `amixer` values and the requirement to persist them to
  `/etc/voicecard/wm8960_asound.state`.

### Changed
- **Transcription is web-initiated** from the web UI, on demand
  (`specs/transcription.md`, `specs/state-machine.md`).
- **Recording control is now shared** between the button and the web UI (FR-23).
- **Diarization added** via OpenAI `gpt-4o-transcribe-diarize`, gated on a configured
  key; writes a separate `transcript_diarized.md` (`specs/transcription.md`,
  `specs/storage.md`).
- **Network framing qualified.** The app now serves a LAN web UI, and diarization needs
  internet; `requirements/connectivity.md` and `non-functional.md` (NFR-1) updated —
  recording and local transcription stay offline.
- **Config additions** (`specs/configuration.md`): `[web]` and `[diarization]` sections.
- **New technical decisions:** TD-5 (two-artifact diarized output) and TD-6
  (named-speaker enrollment) resolved into `web-ui.md`; TD-7 (long-session upload to
  OpenAI) open with an adopted approach pending experiment 0001.
- **Backlog:** B-I1 (Web UI) promoted into the active requirements; B-T5
  (summarization) added as a designed-for future item.
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
  Dropped the Pi Zero 2W removable-device design-intent content from `hardware.md`,
  `transcription.md`, and the systemd capability rationale
  (`CAP_SYS_MODULE`/`CAP_SYS_ADMIN` no longer needed).
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
- **Retired FR-11 from v1.** Dropped the related state-machine path,
  dependencies, and transcription cancellation path.

### Fixed
- **Documentation audit corrections** across the RPi set:
  - **Python/OS mismatch.** The target OS is Debian 13 "trixie" (Python 3.13);
    the docs called for a "Python 3.11 venv" as if 3.11 shipped with it. Clarified
    that 3.11 is the *minimum* and the venv uses the newer OS default
    (`adr/0002-python-venv-over-docker.md`, `specs/install-service.md`).
  - **`hardware.md` RAM table** contradicted itself (Model row "2 GB min" vs. RAM
    row "4 GB"). Collapsed to one row: 2 GB min, 4 GB recommended, 8 GB supported.
  - **Boot config** (`reference/respeaker-2mic-hat.md`) was missing `dtparam=spi=on`
    despite the APA102 LEDs running over SPI; added it to the `config.txt` block.
  - **"Single-threaded"** in `specs/state-machine.md` contradicted its own
    Concurrency table; reworded to "single-threaded control loop" with the
    transcription worker called out.
  - **`min_duration_seconds` semantics** were session-level in `configuration.md`
    but per-chunk in `recording.md`; reconciled to session-level so a short final
    chunk of a longer session is no longer silently dropped.
  - **Minor clarifications:** transcript **Duration** is derived from `session.wav`
    (not "all chunks", which are deleted); FR-18 notes the installer leaves
    `transcription.threads` at its default; the systemd contract notes
    `network.target` is ordering-only (per NFR-1) and that `systemctl restart` is
    the supported way to apply config changes.

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
  recording, storage/filesystem-state, transcription, and the installer + systemd
  service contract. `FR-n` identifiers give the implementation and `earshot-tui`
  stable requirement IDs to trace behavior to.
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
  end); `session.wav` is the transcribed artifact. See the
  [audio storage format ADR](adr/0001-audio-storage-format.md).

### Notes
- Scope is **Pi 4B + ReSpeaker**.
