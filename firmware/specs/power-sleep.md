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
curve, snapped to the nearest **5%**:

| Voltage | 3.20 V | 3.40 V | 3.70 V | 3.90 V | 4.12 V |
| ------- | ------ | ------ | ------ | ------ | ------ |
| Percent | 0%     | 25%    | 50%    | 75%    | 100%   |

Clamps: `≥ 4.12 V → 100%`, `≤ 3.20 V → 0%`. These values are provisional until
measured against real pack behavior on the target board.

## Battery states and policy

Use coarse states with hysteresis:

| State | Enter | Recover | Behavior |
| ----- | ----- | ------- | -------- |
| OK | above LOW recovery | — | normal operation |
| LOW | ≤ 15% estimate | ≥ 20% estimate | visual advisory; do not interrupt active recording |
| CRITICAL | ≤ 5% estimate or pack voltage near safe floor, provisionally ≤ 3.30 V | ≥ 10% estimate | block new recordings; if reached during recording, attempt graceful stop/save |

Policy details:

- **LOW before recording:** recording is allowed.
- **LOW during recording:** show/dismiss advisory without aborting capture.
- **CRITICAL before recording:** block new recording and show a blocking battery
  condition.
- **CRITICAL during recording:** attempt graceful stop/save, commit valid audio if
  possible, then show the blocking battery condition.
- **Automatic power-off:** do not release the VBAT latch automatically in v1 until
  hardware brownout and battery behavior are validated. Prefer preserving data
  and keeping the e-paper warning visible.
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
- **Wake source:** buttons on **BTN_REC (GPIO0)** and **BTN_PWR (GPIO18)**; any
  button press wakes the device.

> The reference firmware used deep sleep. earshot v1 uses **light sleep** after
> the same 120 s with buttons-only wake; the sleep depth is open — see **TD-4**
> in `../requirements/open-technical-decisions.md`.

See also `recording-playback.md` and
`../reference/device-rendering-constraints.md`.
