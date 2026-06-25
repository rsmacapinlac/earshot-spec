# Open Technical Decisions

A living registry of engineering decisions that are deliberately unresolved or
rest on assumptions worth revisiting — the technical counterpart to
`open-ux-questions.md`. Nothing here is settled; each entry notes an interim
default so implementation isn't blocked. When one is decided, fold the outcome
into the relevant ADR/spec and mark the entry **Resolved** (with the date and
where it landed).

Each has a stable ID (`TD-n`). "Owner" names an ADR when one already frames the
decision; standalone entries have no ADR yet.

| ID | Decision | Status | Owner |
| -- | -------- | ------ | ----- |
| TD-1 | Battery gauge accuracy, filtering, and critical thresholds | Open | — |
| TD-2 | Audio down-mix method | Open | — |
| TD-3 | Audio quality ceiling (16 kHz / 16-bit) | Open | — |
| TD-4 | Sleep depth (light vs deep) | Open | — |
| TD-5 | Low-power treatment during long REC/PLAY | Open | — |

---

## TD-1 — Battery gauge accuracy, filtering, and critical thresholds

**Status:** Open · **Related:** `../specs/power-sleep.md` (Battery)

The voltage→percent curve is a generic at-rest Li-ion approximation snapped to
5% — fine for a coarse gauge, not fuel-gauge accurate. The battery is a generic
**3.7 V single-cell LiPo on an MX1.25 connector**; its manufacturer and capacity
(mAh) are unspecified, so there is no datasheet curve and no validated runtime
meaning for percent. The ×2 divider ratio also assumes this board's specific
resistor divider. Load sag during recording/playback/refresh pulls terminal
voltage below the at-rest curve and can cause false-low readings unless filtered
or sampled at rest.

**Out of scope for v1:** true fuel-gauge (coulomb-counting) accuracy, and any
1%-resolution SoC display. This board has only a voltage ADC path — no current
sense and no gauge IC (see ADR-0001) — so fine-grained SoC is not achievable in
software. Percent stays display/debug only at 5% granularity; policy stays on
coarse OK/LOW/CRITICAL states.

Path to closing this item:

- **Divider/ADC calibration:** compare reported voltage to a multimeter reading
  and store a correction factor → makes displayed voltage accurate.
- **Discharge log:** log pack voltage vs. elapsed time from full to empty under
  realistic load, then fit the voltage→percent curve and set LOW/CRITICAL
  thresholds on measured behavior.
- **Filtering / load handling:** EMA or rolling median, with IR compensation or
  rest-sampling so load sag doesn't distort SoC.

**Interim default:** generic at-rest LiPo curve in `../specs/power-sleep.md`,
coarse OK/LOW/CRITICAL states, LOW at ≤15% / recover ≥20%, provisional CRITICAL
at ≤5% or ≤3.30 V / recover ≥10%. With the updated curve these percent triggers
map to higher pack voltages than the old curve (LOW ≈ 3.65 V, CRITICAL ≈ 3.45 V),
with ≤3.30 V as the empty/backstop clamp.

## TD-2 — Audio down-mix method

**Status:** Open · **Related:** `../specs/recording-playback.md`

Capture keeps the **left** channel only. Averaging L+R is moot while the codec is
mono (both slots carry the same mic) but becomes a real choice if multi-mic
hardware (e.g. an ES7210) is ever used.

**Interim default:** left-channel-only (matches the mono hardware).

## TD-3 — Audio quality ceiling

**Status:** Open · **Related:** `../specs/recording-playback.md`

Fixed 16 kHz / 16-bit mono — adequate for voice and speech-to-text. Raising it
costs storage and future transfer time. Revisit only if a use case needs higher
fidelity.

**Interim default:** 16 kHz / 16-bit mono.

## TD-4 — Sleep depth: light vs deep

**Status:** Open · **Related:** `../specs/power-sleep.md`

v1 uses light-sleep after 120 s to measure real battery life; deep sleep is the
fallback if life is poor. The 120 s inactivity trigger and button wake are settled;
a periodic battery-check timer wake (to detect LOW/CRITICAL while asleep) is pending
experiment `../experiments/0001-timer-wake-check.md`. Only the sleep depth is open
here. Decide after measurement.

**Interim default:** light-sleep (v1).

## TD-5 — Low-power treatment during long REC/PLAY

**Status:** Open · **Related:** `../specs/power-sleep.md`

Active states currently run at full power. Whether long recording/playback
sessions warrant any power reduction is unknown until measured on hardware.

**Interim default:** full power while active.
