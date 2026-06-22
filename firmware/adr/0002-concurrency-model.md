# 0002 — Concurrency & Core Allocation

**Status:** Accepted

## Context

The ESP32-S3 is dual-core at 240 MHz. Three things contend for time: button
servicing (must feel instant), the ~300–500 ms blocking panel refresh, and —
once real audio lands — continuous I²S DMA for capture/playback plus SD writes.

## Decision

- **Core 1**: Arduino `loop` handles button polling, the state machine, timers,
  and canvas composition. The current audio worker task is also pinned to core 1
  but runs at a higher FreeRTOS priority than the app loop.
- **Core 0**: the blocking e-paper refresh, on a dedicated FreeRTOS task
  signalled by notification. Display refresh remains isolated from the audio worker.
- The display service exposes a non-blocking `busy()`; the app only composes a
  new frame when no refresh is in flight, and keeps servicing buttons meanwhile.
- **Audio is the real-time citizen.** When recording or playing, the I²S/SD worker
  runs at higher priority than the app loop. The per-second timer's *partial*
  refresh must be cheap and must never block audio. If UI work would risk an
  underrun, skip/defer that tick rather than glitch audio.
- **Shared I²C** is accessed under a single mutex/owner; non-audio I²C work
  happens between audio operations, not during DMA setup.

## Consequences

- Clear ownership: input + logic on one core, blocking/real-time work on the
  other.
- Display refresh and audio are isolated by core in the current firmware: display
  refresh is on core 0; the audio worker is pinned to core 1 at higher priority
  than the app loop. This bounds how much UI work can occur during REC/PLAY
  (acceptable: timers are 1 Hz).
- Canvas is single-writer (core 1) and read by the blit just before hand-off, so
  no canvas locking is needed; only the panel framebuffer crosses cores, guarded
  by the `busy()` handshake.

