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

- **LOW before recording:** recording is not allowed. 
- **LOW during recording:** notify the user that the recording will stop, then stop the recording.
- **CRITICAL is a full lockout ending in deepest sleep.** On entering CRITICAL the
  firmware gracefully stops and commits any active recording (or LABEL_CAPTURE),
  stops any playback/browsing, draws the CRITICAL warning to e-paper, then enters
  the deepest sleep the board supports with the VBAT latch **held**. See
  `state-machine.md` → "Condition & interrupt screens" for the full sequence.
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

- **Inactivity timer (global):** a single inactivity timer (`ULTRA_SLEEP_MS` =
  **120 s**, tracked via `lastActivityMs` / `resetActivity()`) runs in **every
  screen**, not only MAIN. Any button activity resets it. When it expires the device
  draws the SLEEP screen and enters sleep **regardless of the current screen**.
- **Suspended during active audio:** the timer is **stopped while audio is capturing
  or playing** — a clip (RECORDING), a spoken label (LABEL_CAPTURE), or PLAYBACK.
  Neither a capture nor a playback is interrupted by sleep. When the activity **ends**
  (a recording saves, or playback finishes/stops) the timer is **reset** and begins
  counting again.
- **Wake returns to MAIN:** buttons on **BTN_REC (GPIO0)** and **BTN_PWR (GPIO18)**
  wake the device — any button press wakes it. Whatever screen was showing when it
  slept is **not** restored; the device always wakes to the **MAIN** screen. The
  press that wakes it is consumed by waking and is not acted on as an in-state press.
- **No battery sampling while asleep (buttons-only wake).** Nothing wakes the device
  on its own during sleep, so a battery that crosses into LOW or CRITICAL *while
  asleep* is **not** detected autonomously. It is caught on the **next button wake**,
  which re-checks the battery and raises the relevant condition — LOW advisory, or the
  CRITICAL save→warn→deepest-sleep sequence — instead of returning silently to MAIN.
  This is acceptable because captures are always committed *before* the device sleeps,
  so no recording is at risk if the pack browns out during sleep; the only cost is that
  the device may brown out still showing the SLEEP screen rather than a CRITICAL
  warning. The exact brownout behavior is unvalidated — see TD-1.
- **A battery check runs on the wake/boot path.** Because wake is a cold boot
  (ADR-0005), the boot sequence performs a battery read before settling in MAIN. This
  is what raises LOW/CRITICAL after a crossing that happened during sleep, and — since
  a cold boot retains no prior state — it evaluates the level directly rather than as
  an edge (already ≤ LOW → LOW; already ≤ CRITICAL → CRITICAL sequence).
- **CRITICAL sleep and idle sleep are the same depth (deep sleep) but differ in
  trigger and behaviour.** Idle sleep is entered after the 120 s timer and wakes to
  MAIN; CRITICAL is entered **immediately** (not after the timer) and stays locked
  until recovery. Both are deep sleep (ADR-0005) and both hold the VBAT latch — see
  the CRITICAL policy under "Battery states and policy" above and `state-machine.md`
  → "Condition & interrupt screens".

> Like the reference firmware, earshot uses **deep sleep** after the 120 s
> inactivity timer (**ADR-0005**). Wake is **button-driven** — nothing samples the
> battery during sleep — and is a cold boot into MAIN. The latch must be held through
> deep sleep (RTC GPIO hold); battery filter state is lost on sleep and
> re-initialises on wake (harmless, since sleep does no sampling).

See also `recording-playback.md` and
`../reference/device-rendering-constraints.md`.
