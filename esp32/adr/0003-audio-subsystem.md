# 0003 — Audio Subsystem: Raw I²S + Manual ES8311

**Status:** Accepted

## Context

The target board uses a single **ES8311** mono codec for both microphone input
and speaker output. The reference firmware's codec/I²S BSP is not vendored into
earshot. The audio format, I²S transport details, WAV behavior, and playback
contract are specified in `../specs/recording-playback.md`.

Three implementation routes were considered: port the reference BSP, adopt
ESP-ADF, or hand-roll the small amount of codec/I²S setup needed by earshot.

## Decision

Drive audio with the ESP-IDF I²S driver directly and manual ES8311 register
initialization over the shared I²C bus — no ESP-ADF and no opaque board BSP.

Keep codec/I²S details behind a small audio service so the app state machine does
not touch registers, DMA, WAV headers, or raw I²S buffers directly.

## Consequences

- Maximum control and no framework lock-in.
- ES8311 register sequencing and I²S clocking remain likely bring-up/debug areas.
- The codec shares the board I²C bus, so initialization and bus ownership must be
  coordinated with other I²C users.
- Audio format changes should be localized to the audio service and
  `../specs/recording-playback.md`.

## Alternatives

- **Port the reference BSP** — fastest path to known-good behavior, but hides the
  learning, adds opaque abstractions, and is not vendored here.
- **ESP-ADF** — batteries-included pipelines, but heavy, a different build setup
  than the current sketch flow, and overkill for one mono codec.
