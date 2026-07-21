# Earshot (Raspberry Pi)

This is a **product**. It shares the Earshot name and philosophy (local-only capture and local-first
operation) but is a distinct implementation with its own hardware, software stack, and docs.

It is meant to be permanently put on a desk, plugged into power and connected to wifi.

This documentation set is **v1.0** (see [CHANGELOG.md](CHANGELOG.md) and
`../AGENTS.md`). 

## What it does

1. Boots headless as a systemd service; LED pulses **white**, then solid **green** when idle.
2. Press the button → records (LED pulses **red**), rolling over to a new WAV chunk every 15 min.
3. Press again → finalizes the session by concatenating chunks (LED **amber**), then returns to **green**.
4. Manage recordings from a **web UI** at the Pi's IP — browse, listen, delete, start/stop recording, and run transcription (faster-whisper, LED **amber**) on demand. With an OpenAI key, **diarize** a session — replacing its single `transcript.md` with a speaker-labelled version.
5. Hold the button 3 s while idle → safe shutdown.

## Hardware

- **Raspberry Pi 4 Model B**
- **Seeed ReSpeaker 2-Mic Pi HAT** (WM8960 codec, 2 MEMS mics, GPIO17 button, 3× APA102 LEDs)

## Documentation map

| Area | Directory | What lives here |
|---|---|---|
| Product needs & scope | [`requirements/`](requirements/README.md) | Supported hardware, non-functional targets, connectivity, out-of-scope, open questions |
| Technical decisions | [`adr/`](adr/README.md) | Why the major approaches were chosen |
| Exact behavior (normative) | [`specs/`](specs/README.md) | Config schema, state machine, recording/finalization, storage, transcription, install/service |
| Hardware facts | [`reference/`](reference/respeaker-2mic-hat.md) | ReSpeaker 2-Mic HAT hardware facts and expected device configuration |
| Evidence for open decisions | [`experiments/`](experiments/README.md) | Hypothesis-driven hardware validation supporting TDs |
