# Earshot (Raspberry Pi)

This is a **product**. It shares the Earshot name and philosophy (local-only capture and local-first
operation) but is a distinct implementation with its own hardware, software stack, and docs.

It is meant to be permanently put on a desk, plugged into power and connected to wifi.

This documentation set is **v1.0** (see [CHANGELOG.md](CHANGELOG.md) and
`../AGENTS.md`). 

## What it does

1. Boots headless as a systemd service; LED pulses **white**, then solid **green** when idle.
2. Press the button → records (LED pulses **red**), rolling over to a new WAV chunk every 15 min.
3. Press again → finalizes the session by encoding the chunks into one `session.m4a` (LED **amber**), then returns to **green**.
4. Manage recordings from a **web UI** at the Pi's IP — browse, listen, delete, name sessions, start/stop recording, and see live device status.
5. **Transcribe** on demand (LED **amber**) — on the Pi itself, needing no service, key, or internet. Optionally point it at an [earshot processing service](../service/README.md) on your LAN to make that far faster and to add **diarization**, the one thing the Pi can't do alone.
6. Hold the button 3 s while idle → safe shutdown.

## Hardware

- **Raspberry Pi 4 Model B**
- **Seeed ReSpeaker 2-Mic Pi HAT** (WM8960 codec, 2 MEMS mics, GPIO17 button, 3× APA102 LEDs)

## Documentation map

| Area | Directory | What lives here |
|---|---|---|
| Product needs & scope | [`requirements/`](requirements/README.md) | Supported hardware, web UI capabilities, non-functional targets, connectivity, open questions |
| Technical decisions | [`adr/`](adr/README.md) | Why the major approaches were chosen |
| Exact behavior (normative) | [`specs/`](specs/README.md) | Config schema, state machine, recording/finalization, storage, transcription, install/service |
| Hardware facts | [`reference/`](reference/respeaker-2mic-hat.md) | ReSpeaker 2-Mic HAT hardware facts and expected device configuration |
| Evidence for open decisions | [`experiments/`](experiments/README.md) | Hypothesis-driven hardware validation supporting TDs |

Diarization is **not** in this track — it needs an optional
[processing service](../service/README.md), specified separately. Local transcription is
here, in [`specs/processing.md`](specs/processing.md).
