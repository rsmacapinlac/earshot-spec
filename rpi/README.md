# Earshot (Raspberry Pi)

A Raspberry Pi application that records conversations locally. 

This is a **separate product** from the ESP32-based `esp32/` track —
it shares the Earshot name and philosophy (local-only capture, physical offload)
but is a distinct implementation with its own hardware, software stack, and docs.

This documentation set is **v1.0** (see [CHANGELOG.md](CHANGELOG.md) and
`../AGENTS.md`). 

## What it does

1. Boots headless as a systemd service; LED pulses **white**, then solid **green** when idle.
2. Press the button → records (LED pulses **red**), rolling over to a new WAV chunk every 15 min.
3. Press again → concatenates chunks **amber**, returns to **green** after completed.
4. After ~3 min idle, transcribes pending sessions with faster-whisper (LED **amber**).
5. Hold the button 3 s while idle → safe shutdown.

## Hardware (as-built)

- **Raspberry Pi 4 Model B** (running `pi-earshot-pi4`)
- **Seeed ReSpeaker 2-Mic Pi HAT** (WM8960 codec, 2 MEMS mics, GPIO17 button, 3× APA102 LEDs)

## Documentation map

| Area | Directory | What lives here |
|---|---|---|
| Product needs & scope | [`requirements/`](requirements/README.md) | Supported hardware, non-functional targets, connectivity, out-of-scope, open questions |
| Technical decisions | [`adr/`](adr/README.md) | Why the major approaches were chosen |
| Exact behavior (normative) | [`specs/`](specs/README.md) | Config schema, state machine, recording/encoding, storage, transcription, USB offload, install/service |
| Hardware facts | [`reference/`](reference/respeaker-2mic-hat.md) | ReSpeaker 2-Mic HAT + observed device configuration |
| Evidence for open decisions | [`experiments/`](experiments/README.md) | Hypothesis-driven hardware validation supporting TDs |
