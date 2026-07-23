# earshot — spec repository

Source of truth for the **earshot** products and software requirements, firmware/software
specs, architecture decisions, and hardware references. This repo holds documentation
only; each product's implementation lives in its own repository (linked below).

The earshot family of products is a virtual assistant for meetings. It can simply record,
or you can ask it to do more (eg. transcribe, diarize, summarize). It is **local-first**:
audio and transcripts stay on hardware you own, and no feature requires the internet.

## Tracks

Two **devices** capture audio; one **service** turns it into transcripts.

| Track | Kind | What it is | Runs on | Docs version | Implementation |
|---|---|---|---|---|---|
| [`esp32/`](esp32/requirements/README.md) | device | Pocket e-paper **voice-note recorder** firmware | Waveshare ESP32-S3 1.54" e-Paper (ESP32-S3-PICO-1) | v1.7 | [rsmacapinlac/earshot-firmware](https://github.com/rsmacapinlac/earshot-firmware) |
| [`rpi/`](rpi/README.md) | device | Desk **conversation recorder** with a LAN web UI | Raspberry Pi 4B + Seeed ReSpeaker 2-Mic HAT | v1.0 | [rsmacapinlac/earshot](https://github.com/rsmacapinlac/earshot) |
| [`service/`](service/README.md) | service *(optional)* | Faster **transcription**, and **diarization** over HTTP | any Docker host on your LAN | v0.1 (draft) | _not yet created_ |

**The devices work on their own** — a Pi records and transcribes with no service, account,
or internet. The service is an upgrade for anyone who has somewhere to run it: much faster
transcription, and speaker diarization, which is the one thing a recorder genuinely cannot
do. See [the recorder's decision](rpi/adr/optional-processing-service.md) and
[the service's](service/adr/separate-processing-service.md).

Each track is versioned as a whole; see its `CHANGELOG.md`
([esp32](esp32/CHANGELOG.md), [rpi](rpi/CHANGELOG.md), [service](service/CHANGELOG.md))
for history.

## Documentation roles

Every track carries the same subdirectories, each with a distinct purpose (the service
track has no `reference/` or `experiments/` — it owns no hardware):

| Directory | Holds |
|---|---|
| `requirements/` | Product/user needs and cross-cutting qualities — *what* the device must do |
| `specs/` | Normative behavior: exact thresholds, file formats, state transitions, contracts |
| `adr/` | Architecture decision records — *why* a major approach was chosen |
| `reference/` | Non-normative hardware facts and bring-up/history notes |
| `experiments/` | Hypothesis-driven hardware validation that resolves an open decision |

When docs disagree, `specs/` is authoritative for behavior.

## Conventions for contributors (incl. agents)

Repo-wide working conventions — documentation roles, editing rules, and per-track
status — live in [`AGENTS.md`](AGENTS.md). Read it before making changes.
