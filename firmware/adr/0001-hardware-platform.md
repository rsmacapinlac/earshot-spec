# 0001 — Hardware Platform: Waveshare ESP32-S3 1.54" e-Paper

**Status:** Accepted

## Context

The target board is the **Waveshare ESP32-S3 1.54" e-Paper** board, identified in
reference firmware as `S3_ePaper_1_54`, using an ESP32-S3-PICO-1 module.

## Decision

Build earshot v1 specifically for the Waveshare ESP32-S3 1.54" e-Paper board.
Treat this board as the accepted hardware target, not as one interchangeable
backend among many.

The v1 firmware uses the board's integrated:

- ESP32-S3 MCU;
- 200×200 1-bit e-paper display;
- two physical buttons;
- ES8311 mono audio codec for microphone input and speaker output;
- SD_MMC storage in 1-bit mode;
- battery ADC path;
- VBAT software power-hold latch;
- peripheral rail controls for display/audio.

Exact pins and hardware facts live in `../reference/hardware-pinout.md` and the
other files under `../reference/`.

## Consequences

- v1 can optimize for this board instead of carrying generic hardware abstraction
  for unknown future boards.
- Product limitations follow the board: mono capture/playback, two-button input,
  200×200 1-bit display, SD-card storage, and board-specific power/latch
  behavior.
- Specs should describe behavior for this board and reference the hardware
  pinout rather than restating pins everywhere.

