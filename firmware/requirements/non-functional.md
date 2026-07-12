# Non-Functional Requirements

Cross-cutting quality requirements for earshot. The subsystem specs
(`../specs/recording-playback.md`, `../specs/power-sleep.md`) own the concrete numbers;
this document states the goals they serve and the constraints that span
subsystems. Previously referenced from `../specs/recording-playback.md` as "the
broader NFR notes."

## Power & battery (core goal)

- **Battery life is a primary product goal.** The device must sleep aggressively
  when idle: **deep sleep** after 120 s of inactivity (ADR-0005).
- Active states (RECORDING, PLAYBACK, browsing) may run subsystems at full power;
  MAIN must shed everything it can while keeping the VBAT latch held and the
  e-paper image.
- Battery UX and policy must be based on coarse states, not precise percentage:
  OK, LOW, and CRITICAL.
- LOW battery warns the user but must not abort an active recording.
- CRITICAL battery must protect recordings and conserve remaining charge: stop and
  save any active capture, block further activity, show the warning, and drop to the
  lowest-power sleep — rather than continuing to drain.
- Battery readings must be filtered/hysteretic enough to avoid false state
  changes from brief load sag during audio, SD writes, or display refresh.
- The device is powered only while software holds the VBAT latch; the firmware
  must never drop the latch unintentionally. v1 must not automatically release
  the latch on low battery until hardware behavior is validated.

## Responsiveness

- Button presses must feel immediate even though a panel refresh busy-waits
  ~300–500 ms. Input servicing must not be blocked by display refresh; see
  `../adr/0002-concurrency-model.md`.
- Live timers (REC/PLAY) update once per second via partial refresh; this must
  not glitch or stall audio.

## Display fidelity

- All UI must render within the 1-bit GFX constraints in
  `../reference/device-rendering-constraints.md`: discrete font sizes, ASCII only, no
  letter-spacing, FreeSans proportional widths, 192 px content width.
- No animation, motion, or grayscale. Full refresh only on screen change;
  partial refresh for isolated live fields. Periodic cleanup full-refresh is not
  part of the current baseline and should be added only if panel testing requires it.

## Audio quality & integrity

- Voice-grade capture: 16 kHz, 16-bit mono PCM in WAV (`../specs/recording-playback.md`).
  Adequate for speech and future transcription; not music-grade by design.
- A recording must be written durably: a power loss or cancel must not leave a
  corrupt note that breaks the list. Sub-~1000-byte captures are discarded.

## Storage & scale

- The recordings list is built by **scanning the SD card**, then the current UI
  caches the newest 16 notes and pages the 3-row window over that bounded set
  (`../specs/storage.md`).
- Each note carries sidecar metadata (duration) and may carry an optional
  voice-label sidecar. Storage is the source of truth; RAM holds the bounded UI
  cache.

## No time/date dependency

- Recording, labeling, listing, playback, deletion, and sync must not require a
  valid clock, date, timezone, RTC, NTP, or manual time setup.
- Recording identity is the stable `rec-NNNNNN` ID plus optional voice label, not
  date/time metadata.

## Maintainability & portability

- The firmware is layered (`hal/` → `drivers/` → `app/`) so subsystems can be
  brought up and tested independently. The current `earshot/` tree is a UX
  prototype and is fully reworkable; only the display/rendering layer is worth
  preserving as-is.
- Hardware access goes through HAL/driver seams so the board pin map and codec
  details stay in one place (`../reference/hardware-pinout.md`).

## Forward compatibility

- A **transfer/sync seam** must exist in v1 even though no sync ships, so a future
  companion-app sync can be added without reworking storage or the state machine.

See also `../adr/README.md` for the decisions that satisfy these requirements.
