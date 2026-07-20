# 0003 — Hardware Abstraction Layer

**Status:** Accepted

## Context
The application depends on Pi-specific hardware. Without abstraction, the full
app cannot run or be tested on a development machine, and supporting hardware
variants would scatter conditionals through the codebase.

| HAT | Button | LED control | ALSA card | Speaker |
|---|---|---|---|---|
| ReSpeaker | GPIO17 | APA102 via SPI | `seeed-2mic-voicecard` | No |

## Decision
Hardware-specific components sit behind interfaces, selected at startup from
`hardware.hat` in `config.toml`:

- `ButtonInterface` — press and hold detection
- `LEDInterface` — colour and pattern
- `AudioCaptureInterface` — microphone capture
- `DisplayInterface` — renders state on an LCD; **no-op on the ReSpeaker HAT**

Each interface has a **Real** implementation (Pi hardware: GPIO/SPI/ALSA) and a
**Stub** (in-memory/no-op) for local development and testing. `AudioOutputInterface`
(speaker) is planned for v2.

## Consequences
- Full application logic and the encoding pipeline can be developed and tested
  locally without a Pi (Stub HAL).
- Adding a HAT requires only new implementations — no application-logic changes.
- The active HAT is chosen by config, not runtime detection, keeping startup simple.
- Integration testing on real hardware still requires a Pi.

> **As-built:** the running code ships two HAL backends only — `pi` (ReSpeaker:
> `PiButton` on GPIO17, APA102 LED over SPI, ALSA capture) and `stub`. No LCD/
> Whisplay display backend exists in the code; on the ReSpeaker the display is a
> no-op and the **LED is the sole feedback channel**.
