# 0001 — Timer-Wake-to-Check Battery Sampling During Sleep

**Status:** Proposed (not yet run) · **Type:** hardware validation (non-normative)
**Related:** `../specs/power-sleep.md` (Sleep, Battery states & policy),
`../specs/state-machine.md` (LOW/CRITICAL condition screens),
`../requirements/open-technical-decisions.md` (TD-1)

**Decision this supports:** whether to add a **single fixed-interval** timer wake
(~5 min, `SLEEP_BAT_CHECK_MS`) to sleep so LOW/CRITICAL are detected autonomously
while asleep, instead of the current buttons-only wake. The data must confirm the
fixed interval (a) never misses or skips a transition, (b) catches CRITICAL within
the latency budget, and (c) costs less than the power ceiling. Relates to TD-1
(battery checks/thresholds) and informs TD-4 (sleep depth) without resolving it.

**Decision rule:** adopt the fixed ~5-min sleep poll if it satisfies **all** of the
success criteria below. If the cost exceeds the ceiling, lengthen the interval (the
latency budget allows it) and re-evaluate; if a coarse interval risks skipping
transitions (Test E), fall back to CRITICAL-only polling or button-wake detection.

## Assumptions

These shape the experiment and its pass/fail lines. If one proves false, the
fixed-interval decision reopens.

- **A1 — CRITICAL latency budget:** detecting CRITICAL within **~5 min** of the
  threshold crossing while asleep is acceptable. This sets the poll interval ceiling.
- **A2 — Unattended sleep duration:** a device realistically sits asleep on the
  order of **hours**, not days. (Days would make baseline sleep current dominate and
  could change the calculus.)
- **A3 — Cost ceiling:** the sleep-poll average-current adder must be **< 5%** of
  baseline sleep current.
- **A4 — Slow sleep discharge:** because sleep current is small, the pack voltage
  moves slowly during sleep, so a single ~5-min poll cannot skip a LOW or CRITICAL
  transition. *Assumed, and validated by Test E.*
- **A5 — Display is effectively free:** e-paper is bistable, so showing LOW/CRITICAL
  costs one refresh per transition and then persists at zero power; the only
  recurring cost is the poll itself.
- **A6 — Fixed beats adaptive here:** because A1's 5-min budget makes the poll cheap
  even when run continuously, an adaptive (slow/fast tier) scheme would save only
  ~microamps and is not worth the added firmware complexity. Adaptive is the
  fallback to revisit only if A1 tightens substantially.

## Background

Today's Sleep spec lists **buttons-only** wake sources, so the awake
`BAT_CHECK_INTERVAL_MS` (30 s) re-check does not run while asleep. The proposed
change adds an ESP32 timer wake (`esp_sleep_enable_timer_wakeup`) at a separate,
coarser sleep interval `SLEEP_BAT_CHECK_MS` (~5 min): the device wakes briefly,
samples the battery, and either re-sleeps (state unchanged) or raises the relevant
condition (LOW advisory, or the CRITICAL save→warn→deepest-sleep sequence).

## Hypotheses

- **H1 (correctness):** every induced OK→LOW and LOW→CRITICAL transition during
  sleep is detected on a timer wake and produces the correct behavior; none missed.
- **H2 (latency):** CRITICAL detection latency ≤ `SLEEP_BAT_CHECK_MS` + one sample
  window (within the ~5 min budget, A1).
- **H3 (cost):** average current adder < 5% of baseline sleep current (A3).
- **H4 (robustness):** filtering prevents false transitions from ADC noise or
  transient sag (zero false transitions).
- **H5 (no skipped transition):** measured worst-case sleep discharge × one interval
  is small enough that no LOW/CRITICAL crossing is stepped over (validates A4).

## Equipment

- Waveshare ESP32-S3 1.54" e-Paper board (target hardware, ADR-0001).
- **Bench DC power supply** on the battery rail (set to the post-divider pack
  voltage) — to cross thresholds at exact voltages on demand. Primary stimulus.
- **Current meter** for low-current + transient capture (e.g. Nordic PPK2, or a
  precision shunt + scope). A plain DMM in series gives averages only (fallback).
- USB-serial for the debug log.
- A real 3.7 V LiPo (the MX1.25 cell) — **required** for the discharge-rate test
  (Test E) and one confirmatory pass; the bench supply can't reproduce real drain.

## Firmware instrumentation (debug build)

- Add the sleep timer wake at `SLEEP_BAT_CHECK_MS`, **overridable** for the run
  (e.g. 2 / 5 / 10 min).
- On every wake, log: `t_ms, wake_reason {timer|button}, raw_mV, filtered_mV,
  computed_pct, state {OK|LOW|CRITICAL}, action {none|advisory|critical-seq}`.
- Toggle a GPIO **high while the CPU is awake for a check** so the current meter /
  scope can mark wake windows and measure their duration.
- Do not draw the e-paper on an unchanged state (refresh only on transition).

## Data to collect

| Quantity | Unit | How | Decides |
| --- | --- | --- | --- |
| Baseline sleep current | µA | meter, buttons-only, steady OK voltage | cost denominator (A3); go/no-go |
| Per-wake awake duration | ms | GPIO marker on scope/PPK | per-wake energy |
| Current during a check | mA | meter over the marked wake window | per-wake energy |
| Sleep discharge rate, by band | mV/min | real LiPo, long log, upper band + near knee | H5 / validates A4 |
| Detection latency | s | log: `raised_time − crossing_time` | H2 |
| Detected / missed / skipped | count | Test A logs | H1 / H5 |
| False transitions under noise/sag | count | Test D logs | H4 |
| One-time refresh energy | mJ | meter over a single transition draw | sanity-check A5 |

Derived: average adder = per-wake energy ÷ interval; cost % = adder ÷ baseline;
worst-case drop = discharge rate × interval.

## Procedure

### Test A — Detection correctness (H1)
1. Boot, let the device sleep; set input to OK (above LOW recovery).
2. Lower input below LOW enter; confirm the next timer wake detects LOW and takes
   over sleep (advisory drawn / persists).
3. Lower below CRITICAL enter; confirm the save→warn→deepest-sleep sequence.
4. Raise input above recovery; confirm recovery to OK/IDLE.
5. ≥10 cycles. Record detected vs. missed vs. skipped transitions.

### Test B — Detection latency (H2)
1. Step input across CRITICAL at a logged time, starting from LOW.
2. Compute (condition-raised − crossing) from the log.
3. ≥10 crossings at the chosen interval. Report min/median/max; confirm ≤ budget.

### Test C — Power cost (H3)
1. Hold a steady OK voltage (no transitions).
2. Baseline: average current with timer wake **disabled** (buttons-only), ≥10 min.
3. Repeat with the poll **enabled** at 2 / 5 / 10 min.
4. From the GPIO marker, record per-wake duration and instantaneous current.
5. Compute average-current adder and % vs baseline; project battery-life impact.

### Test D — False-transition rejection (H4)
1. Hold input just above the LOW threshold (within noise margin).
2. Inject transient sag during sample windows.
3. ≥30 windows. Count spurious LOW/CRITICAL transitions (expect 0).

### Test E — Sleep discharge rate (H5) — **real LiPo**
1. On the real cell, log filtered voltage over a long sleep run (hours) at a fixed
   short interval, in both the upper band and near the knee (steeper there).
2. Compute worst-case mV/min × interval = worst-case drop per poll; confirm it is
   well below the LOW→CRITICAL voltage gap so no crossing is skipped.

## Success criteria

- **H1:** 100% of induced transitions detected; 0 missed; 0 skipped.
- **H2:** CRITICAL latency ≤ interval + one sample window (~32 ms), within ~5 min.
- **H3:** average current adder < 5% of baseline at the chosen interval.
- **H4:** 0 false transitions.
- **H5:** worst-case drop per interval ≪ the LOW→CRITICAL voltage gap.

## Risks & confounders

- **Bench supply ≠ battery:** no internal resistance / sag. Use it for deterministic
  crossings; use the real LiPo for Test E and a confirmatory pass.
- **Divider not yet calibrated (TD-1):** absolute voltages may be slightly off; work
  in reported-mV terms and back out true volts after calibration.
- **Curve steepness near the knee:** mV/min accelerates near empty even at constant
  current — Test E must sample there, not just in the flat upper band.
- **Sleep depth:** this experiment fixes one sleep mode; cross-mode cost is TD-4.
- **e-paper refresh** must not be counted into per-check cost — verify no refresh on
  unchanged-state wakes.

## Outcome

_To be filled in after running:_ chosen `SLEEP_BAT_CHECK_MS`, measured baseline
current, per-wake energy, discharge rate, and latency; whether the fixed sleep poll
is adopted in `power-sleep.md` (and which fallback if not); follow-ups for TD-1.
