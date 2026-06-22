# Firmware Bring-up Notes

Non-normative implementation notes preserved from the old implementation plan.
Product requirements live in `../requirements/`, implementation contracts in
`../specs/`, and architectural decisions in `../adr/`.

## Intended layering

```text
earshot/
  earshot.ino
  src/
    hal/        pins, power/latch, I2C owner, SD bus, raw ADC
    drivers/    e-paper BSP, ES8311, I2S audio link, optional RTC reference code
    services/   display, buttons, config scaffold, battery, storage, audio, sleep, transfer seam
    app/        model, state machine, screens, setup/tick orchestration
```

Key seams:

- **Display:** one canvas owner; full refresh on screen changes; partial refresh
  for isolated live fields; app never blocks on panel refresh.
- **Buttons:** OneButton instances dispatch typed BOOT/PWR short/long/double
  events to the state machine.
- **Storage:** SD card is the source of truth; app scans note summaries and keeps
  a bounded newest-16 UI cache.
- **Audio:** state machine calls an app-facing service; only audio/storage layers
  know I2S, ES8311, WAV, temp files, and metadata commit details.
- **No time/date dependency:** recording identity, storage, UI list order, labels,
  and sync seams do not require RTC, NTP, timezone, or boot-time configuration.
- **Power:** rail/latch/sleep operations are centralized; v1 defaults to light
  sleep while sleep depth remains an open technical decision.
- **Conditions:** hardware/system conditions are raised separately from the core
  navigable states; priority should be deterministic.

## Bring-up checklist

1. **Power/latch safety** — serial boot log first; verify GPIO17 latch semantics
   and EPD/audio rail active levels with a meter.
2. **Display** — full/partial refresh smoke test, polarity, ghosting, and button
   responsiveness during refresh.
3. **Buttons** — log short/long/double events and verify thresholds on both
   buttons.
4. **SD_MMC/storage** — mount/unmount tests, `/recordings` creation/scanning,
   free-space reporting, corrupt/orphan recovery.
5. **I2C/audio devices** — scan for codec/control devices needed by v1; verify
   shared-bus locking.
6. **Battery ADC** — compare ADC-derived voltage to a multimeter across charge
   levels and active loads; record TD-1 data.
7. **Audio** — ES8311 register-init smoke test, capture raw buffers, record WAV,
   inspect header/data, play known WAV, watch for I2S underruns.
8. **Concurrency** — REC/PLAY with timer partial refreshes; skip/defer UI refresh
   if audio underruns appear.
9. **Sleep/wake** — 120 s IDLE timeout, SLEEP screen retention, light-sleep
   current, either-button wake, repeated cycles.
10. **Fault injection** — no SD, full SD, low/critical battery simulation,
    corrupt notes, power loss/cancel mid-record.

## Validation refinements still needed

- Deterministic condition priority table, especially NO SD + STORAGE FULL + LOW
  BATTERY / CRITICAL BATTERY + CHARGING.
- Exact orphan/corrupt/temp-file recovery policy.
- Minimum-free-space threshold before recording and behavior when free space
  disappears mid-recording.
- Battery-life target that determines whether TD-4 moves from light sleep to deep
  sleep.
- Hardware validation of whether long REC/PLAY needs a cleanup full-refresh
  cadence.
- Charger-present detection mechanism, if CHARGING should auto-raise.

## Risk notes

| Risk | Mitigation |
| --- | --- |
| ES8311 raw I2S/manual init fails or is noisy | Bring up in isolation; keep register code encapsulated; log I2S errors; use known-good WAV/header tests. |
| Audio worker and app loop contend | Audio worker runs at higher priority; defer/skip UI work under audio load; measure underruns. |
| SD write interruption corrupts notes | Temp file + commit discipline; header backfill before metadata commit; robust scan ignoring incomplete files. |
| Battery gauge inaccurate or load-sensitive | Treat as provisional; measure against pack voltage; use coarse battery states and filtering. |
| Light-sleep current insufficient | Keep power seam swappable; collect current/wake-latency data for TD-4. |
| VBAT latch release mistake powers off device | Centralize latch writes; avoid release calls in v1 unless intentional and hardware-verified. |
| CHARGING cannot auto-trigger | Keep as force-test/future behavior until a board mechanism is defined. |

## Historical implementation notes

The old implementation plan included dated checkpoints from early firmware work.
Useful current conclusions from that history:

- Early RTC/time-bootstrap work is superseded by the no-time/date-dependency v1
  direction.
- Storage moved from flat `/notes` and timestamp/unset directories toward the
  current `/recordings/rec-NNNNNN/` contract in `../specs/storage.md`.
- ES8311/I2S bring-up was the major hardware-facing blocker and should remain
  isolated behind the audio service.
