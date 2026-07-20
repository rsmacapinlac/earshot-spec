# 0005 — Idle Sleep Depth: Deep Sleep

**Status:** Accepted (2026-07-12) · Resolves **TD-4**.

## Context

After 120 s of inactivity the device sleeps (the global inactivity model in
`../specs/power-sleep.md`). The ESP32-S3 offers two modes for this:

- **Light sleep** — CPU paused, RAM and peripheral state retained; execution
  resumes in place. Fast, seamless wake; higher standby current.
- **Deep sleep** — most of the SoC powers down and RAM is lost except RTC slow
  memory; wake is a full reboot that runs `setup()`. Lowest standby current.

v1 had provisionally used light sleep (TD-4) specifically to measure real battery
life before committing. Two facts settle it earlier than a measurement campaign
would: standby drain dominates total battery life for a device that spends most of
its time idle, and the product model already **boots directly into MAIN** and — per
the v1.7 sleep rework — **always wakes to MAIN** without restoring the prior screen.
A cold-boot wake is therefore behaviourally identical to a normal wake.

## Decision

Idle sleep uses **deep sleep**. Chosen for best standby battery life; the
boot-to-MAIN model makes a cold-boot wake a non-issue, and it unifies idle and
CRITICAL sleep to the same depth (they already differ only in trigger and wake
behaviour, not depth).

Wake sources are the two buttons — **BTN_REC (GPIO0)** and **BTN_PWR (GPIO18)**,
both RTC-capable — via the EXT1/GPIO wake already used for sleep (ADR-0004). Sleep
does no battery sampling: wake is buttons-only (`../specs/power-sleep.md`).

## Consequences

- **Wake is a cold boot.** On wake the SoC reboots and re-initialises display, SD,
  audio, and state, then lands in MAIN. Wake is slower and more visible than light
  sleep's instant resume (a full e-paper refresh), which is acceptable given the
  boot-to-MAIN model.
- **The VBAT latch must be held through deep sleep.** GPIO17 must be pinned with RTC
  GPIO hold so it stays HIGH while the drivers are powered down, or the board powers
  off. The CRITICAL path already relies on this mechanism.
- **RAM/filter state is lost each sleep — but this is benign here.** Battery
  EMA/hysteresis history does not survive deep sleep. Because sleep samples no
  battery (buttons-only), the filter simply re-initialises on the cold-boot wake and
  the awake re-check rebuilds it; the repeated-crossing requirement keeps a single
  noisy first reading from flipping state. No RTC-memory retention is required.
- **Idle and CRITICAL sleep are now one depth.** Both are deep sleep; they differ
  only in *when* they are entered (idle after 120 s vs CRITICAL immediately) and in
  *wake behaviour* (idle wakes to MAIN; CRITICAL re-checks the battery and stays
  locked until recovery ≥10%).

## Alternatives

- **Light sleep** (the prior interim default) — simpler, instant stateful resume, no
  latch-hold or re-init concerns, but materially higher standby current. Rejected on
  battery life; kept only as the historical interim while TD-4 was open.
- **Defer until a discharge campaign** — the original TD-4 plan. Rejected: the
  boot-to-MAIN model removes the only real downside of deep sleep (wake behaviour),
  so the decision no longer depends on the measurement.
