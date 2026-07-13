# Changelog

All notable changes to the earshot **firmware documentation** (specs, requirements,
ADRs, reference, experiments). The documentation set is versioned as a whole; the
current version is recorded in `../AGENTS.md`. Dates are ISO-8601 (YYYY-MM-DD).

## [Unreleased]

### Changed
- **Battery gauge/policy split** (`specs/power-sleep.md`,
  `specs/state-machine.md`, `requirements/open-technical-decisions.md`): adopted a
  CrossInk-style LiPo voltage polynomial for 5%-snapped display/debug percentage,
  while making LOW/CRITICAL policy explicitly voltage-backed (LOW ≤3.65 V / recover
  ≥3.70 V; CRITICAL ≤3.45 V / recover ≥3.60 V, with ≤3.30 V as the absolute empty
  backstop). Display rounds VBAT ≥4.10 V up to 100% for practical full-charge UX.
  This keeps user-facing percent as an estimate and safety behavior tied to measured
  VBAT.

## [1.7] — 2026-07-12

### Fixed
- **LOW battery policy corrected to advisory-only** (`specs/power-sleep.md`): the
  LOW policy bullets — present since the v1.5 reorg (`9fc5f49`) — read "recording
  not allowed / stop the recording", contradicting `specs/state-machine.md` (the
  behavioral source of truth) and the design screens, which keep LOW as a
  non-blocking, dismissable advisory with recording continuing. The bullets now
  match: recording is allowed while LOW, and an active capture is never aborted by
  it.

### Changed
- **Sleep is now a global inactivity model** (`specs/power-sleep.md`,
  `specs/state-machine.md`, `reference/firmware-bring-up.md`): the 120 s
  `ULTRA_SLEEP_MS` timer runs in **every** screen, not only the home screen, so the
  device sleeps from whatever screen it is on after inactivity (closes the
  "left in a browsing/confirm screen forever" drain). The timer is **suspended during
  active audio** — RECORDING, LABEL_CAPTURE, and PLAYBACK are never interrupted by
  sleep — and **resets** when the activity ends (a recording saves, or playback
  finishes/stops). On wake, any button returns the device to **MAIN** regardless of
  the screen it slept from; the prior screen is not restored and the wake press is
  consumed. The CRITICAL immediate-deep-sleep path is unchanged.
- **Renamed the `IDLE` state/screen to `MAIN`** across specs, requirements,
  reference, and experiments. Lowercase "idle sleep"/"inactivity" wording is
  unchanged; prior CHANGELOG entries retain the historical `IDLE` name.
- **Sleep is buttons-only** (`specs/power-sleep.md`, `specs/state-machine.md`,
  `requirements/open-technical-decisions.md` TD-4): nothing samples the battery while
  asleep, so a LOW/CRITICAL crossing that happens during sleep is caught on the next
  button wake rather than autonomously. Accepted as low-risk because captures are
  always committed before sleep, so a brownout during sleep endangers no recording.

- **Idle sleep depth resolved to deep sleep** (`adr/0005-idle-sleep-depth.md` — new;
  `specs/power-sleep.md`, `specs/state-machine.md`, `requirements/non-functional.md`,
  `reference/firmware-bring-up.md`): **TD-4 resolved**. v1 no longer uses light sleep —
  idle sleep is deep sleep, chosen for standby battery life; the boot-to-MAIN model
  makes cold-boot wake a non-issue and unifies idle/CRITICAL sleep to one depth. Wake
  is a cold boot into MAIN; the VBAT latch is held through deep sleep; battery filter
  state is lost on sleep and re-initialises on wake (harmless under buttons-only).

### Added
- **ADR-0005 — Idle sleep depth: deep sleep** (`adr/0005-idle-sleep-depth.md`):
  records the light-vs-deep decision, its rationale, and consequences; resolves TD-4.

### Removed
- **Experiment 0001 — Timer-Wake-to-Check** (`experiments/0001-timer-wake-check.md`):
  removed. It validated a periodic battery-check timer wake during sleep; with the
  buttons-only decision above there is no autonomous sleep check to validate, so the
  experiment no longer supports a live decision.

## [1.6] — 2026-06-24

### Changed
- **Rendering constraint** (`reference/device-rendering-constraints.md`): banned the
  5×7 font for any on-screen text (9pt floor); captions/positions use FreeSans 9pt,
  mixed case, with icons carrying the category.
- **Battery gauge curve** (`specs/power-sleep.md`): replaced the 5-point
  voltage→percent approximation with a 7-point generic 3.7 V single-cell LiPo
  **at-rest** curve, still snapped to 5%; moved the empty clamp from ≤3.20 V to
  ≤3.30 V; noted it under-reports under load until a real discharge log exists. As a
  consequence the percent-based triggers now map to higher pack voltages
  (LOW ≈ 3.65 V, CRITICAL ≈ 3.45 V), with ≤3.30 V as the empty/backstop clamp.
- **LOW BATTERY** (`specs/state-machine.md`): documented as **edge-triggered** (once
  on OK→LOW, not continuous); added the **SLEEP exception** — if it fires while
  asleep it takes over the sleep screen and wakes the display, and dismiss (PWR) →
  IDLE (not back to sleep), restarting the 120 s timer.
- **CRITICAL BATTERY** (`specs/state-machine.md`, `specs/power-sleep.md`,
  `requirements/non-functional.md`): now a **full lockout ending in deepest sleep** —
  gracefully stops/commits any active RECORDING or LABEL_CAPTURE, stops
  playback/browsing, draws the warning to the bistable e-paper, then enters the
  deepest sleep with the **VBAT latch held**. A button wake re-checks the battery and
  returns to IDLE only on recovery (≥10%). The v1 "no automatic power-off" decision is
  retained (deep sleep, not latch release); the non-functional requirement was aligned
  to "protect recordings and conserve remaining charge."
- **Sleep wake source** (`specs/power-sleep.md`, `requirements/open-technical-decisions.md`
  TD-4): clarified that detecting LOW/CRITICAL *while asleep* needs a periodic
  battery-check timer wake (in addition to button wake), under evaluation in
  experiment 0001; until adopted, those conditions surface on the next button wake.
- **TD-1** (`requirements/open-technical-decisions.md`): reframed — battery
  identified as a generic 3.7 V LiPo on an MX1.25 connector (manufacturer/capacity
  unspecified); **true fuel-gauge accuracy and 1% resolution declared out of scope**
  (voltage-ADC path only, no current sense / gauge IC — ADR-0001); added a concrete
  path to closing (divider/ADC calibration → discharge log → filtering/load handling).

### Added
- **Label-first playback** (`specs/recording-playback.md`): a labelled note plays
  its spoken `label.wav` first, then the recording, as one continuous playback; the
  progress bar scales to the active segment, and playback falls back to the recording
  if the label is missing or fails to open.
- **Experiments** documentation role (`AGENTS.md`): hypothesis-driven hardware
  validation plans that collect data to drive a design decision, located under
  `firmware/experiments/` and named `NNNN-slug.md`.
- **Experiment 0001 — Timer-Wake-to-Check** (`experiments/0001-timer-wake-check.md`):
  validates a fixed ~5-min timer-wake battery check during sleep (so LOW/CRITICAL are
  detected while asleep). Includes documented assumptions (A1–A6), the data to
  collect, test procedure, success criteria, and a decision rule.
- **Experiments scaffolding** (`experiments/README.md`, `experiments/TEMPLATE.md`):
  an index of experiments and a starter template carrying the Decision / Assumptions /
  Data-to-collect / Decision-rule sections by default.

## [1.5] — baseline

- Reorganized firmware documentation into requirements / specs / ADR / reference
  (commits `9fc5f49`, `486bdcb`). Treated as the prior baseline; see git history for
  detail.
