# Power & Sleep Spec

Battery, power-rail, and sleep contract for earshot, adapted for the Waveshare
ESP32-S3 1.54" e-Paper board.

## Battery measurement

- **Sense pin:** GPIO4 (ADC1_CH3), attenuation `ADC_11db`.
- **Raw reading:** `analogReadMilliVolts`, averaged over **16 samples** spaced
  **2 ms** apart, then ×2 for the resistor divider → pack voltage.
- `v ≤ 0.1 V` is invalid / no reading.
- Battery state should be based on filtered voltage, not a single raw read:
  - keep the 16-sample average for each measurement;
  - apply temporal filtering such as an EMA or rolling median;
  - require repeated threshold crossings where practical before changing state.

## Battery gauge

The ADC voltage gauge is approximate. Do not treat percentage as a precise fuel
gauge. UI and policy should rely primarily on coarse battery states.

A rough percent may still be computed for display/debug using this piecewise
curve, snapped to the nearest **5%**. It approximates the at-rest discharge curve
of a generic 3.7 V single-cell LiPo (the target battery is a 3.7 V LiPo on an
MX1.25 connector):

| Voltage | 3.30 V | 3.60 V | 3.70 V | 3.80 V | 3.90 V | 4.00 V | 4.12 V |
| ------- | ------ | ------ | ------ | ------ | ------ | ------ | ------ |
| Percent | 0%     | 10%    | 25%    | 50%    | 70%    | 90%    | 100%   |

Clamps: `≥ 4.12 V → 100%`, `≤ 3.30 V → 0%`. Interpolate linearly between anchors,
then snap to 5%. This is a generic **at-rest** curve: under recording, playback,
or e-paper refresh the pack sags below these values, so it will under-report SoC
under load until a real discharge log is captured (see TD-1). Capacity (mAh) is
unspecified for this cell, so percent has no validated runtime meaning yet.

## Battery states and policy

Use coarse states with hysteresis:

| State | Enter | Recover | Behavior |
| ----- | ----- | ------- | -------- |
| OK | above LOW recovery | — | normal operation |
| LOW | ≤ 15% estimate | ≥ 20% estimate | visual advisory; do not interrupt active recording |
| CRITICAL | ≤ 5% estimate (primary), or the ≤ 3.30 V empty clamp as an absolute backstop | ≥ 10% estimate | full lockout → stop & commit active capture, block all activity, warn, then deepest sleep with latch held (see policy below) |

Policy details:

- **LOW before recording:** recording is allowed.
- **LOW during recording:** show/dismiss advisory without aborting capture.
- **CRITICAL is a full lockout ending in deepest sleep.** On entering CRITICAL the
  firmware gracefully stops and commits any active recording (or LABEL_CAPTURE),
  stops any playback/browsing, draws the CRITICAL warning to e-paper, then enters
  the deepest sleep the board supports with the VBAT latch **held**. See
  `state-machine.md` → "Condition & interrupt screens" for the full sequence.
- **No new activity while CRITICAL:** recording, playback, and browsing are all
  blocked. A button (or future charger) wake re-checks the battery and returns to
  deep sleep until it recovers (≥10% estimate), at which point normal operation
  resumes at IDLE.
- **No automatic power-off in v1:** the VBAT latch is **not** released
  automatically; CRITICAL minimises drain via deep sleep instead, keeping the
  e-paper warning visible. Whether CRITICAL should ever release the latch (true
  power-off) remains open pending brownout validation — see TD-1.
- Re-check battery at least every **30 s** (`BAT_CHECK_INTERVAL_MS`). More
  frequent checks are allowed around state transitions or active recording if
  they do not affect audio reliability.

> Gauge accuracy, critical thresholds, and load-sag behavior are open hardware
> validation items — see **TD-1** in `../requirements/open-technical-decisions.md`.

## Charger detection

CHARGING is reserved for a future charger-present signal/API. Current firmware
must not auto-raise CHARGING unless a reliable board mechanism is defined.

## Power rails & latch

The e-paper and audio controls are **active-low** peripheral rail enables; the
GPIO map is in the board reference (`../reference/hardware-pinout.md`). The
**VBAT latch** (GPIO17) is not treated as a normal active-low rail: **HIGH holds
board power** and LOW releases the latch for intentional shutdown. The firmware
must assert the latch deliberately and never drop it unintentionally, or the
board powers off.

## Sleep

- **Trigger:** sleep after **120 s** of inactivity (`ULTRA_SLEEP_MS`), tracked via
  `lastActivityMs` / `resetActivity()`.
- **Entry sequence:** show sleep screen → stop transfer server if present → mute
  audio → keep VBAT latched → arm wake → enter sleep.
- **Wake source:** buttons on **BTN_REC (GPIO0)** and **BTN_PWR (GPIO18)** — any
  button press wakes the device. Detecting battery-state transitions *while asleep*
  (the LOW "take over sleep" and CRITICAL sequences in `state-machine.md`)
  additionally requires a periodic **battery-check timer wake**; that mechanism and
  its interval are under evaluation in
  `../experiments/0001-timer-wake-check.md`. Until it is adopted, LOW/CRITICAL are
  surfaced on the next button wake rather than autonomously during sleep.
- **CRITICAL deep sleep is separate from the idle sleep above.** The 120 s sleep
  here is the standard idle sleep (light sleep in v1, depth open per TD-4). CRITICAL
  battery instead enters the **deepest sleep the board supports immediately** (not
  after the 120 s timer), with the VBAT latch held — see the CRITICAL policy under
  "Battery states and policy" above and `state-machine.md` → "Condition & interrupt
  screens".

> The reference firmware used deep sleep. earshot v1 uses **light sleep** after
> the same 120 s; wake is button-driven today, with a battery-check timer wake
> pending experiment 0001. The sleep depth is open — see **TD-4** in
> `../requirements/open-technical-decisions.md`.

See also `recording-playback.md` and
`../reference/device-rendering-constraints.md`.
