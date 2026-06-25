# Changelog

All notable changes to the earshot **firmware documentation** (specs, requirements,
ADRs, reference, experiments). The documentation set is versioned as a whole; the
current version is recorded in `../AGENTS.md`. Dates are ISO-8601 (YYYY-MM-DD).

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
