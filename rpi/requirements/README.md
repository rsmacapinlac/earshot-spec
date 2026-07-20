# Earshot (RPi) — Requirements Index

Earshot is a Raspberry Pi 4B application that records audio via the Seeed
ReSpeaker 2-Mic HAT and stores recordings locally.

Requirements here describe **product/user needs and cross-cutting qualities**.
Exact behavior, thresholds, and contracts live in [`../specs/`](../specs/README.md).
Rationale for major choices lives in [`../adr/`](../adr/README.md).

## Documents

| File | Description |
|---|---|
| [hardware.md](hardware.md) | Supported SBC and HAT — specs and capabilities |
| [non-functional.md](non-functional.md) | Performance, resilience, and out-of-scope items |
| [connectivity.md](connectivity.md) | WiFi is for SSH access only; no application network dependency |
| [open-technical-decisions.md](open-technical-decisions.md) | Deferred engineering decisions (TD-n), with interim defaults |
| [open-ux-questions.md](open-ux-questions.md) | Deferred interaction/UX decisions (UX-n) |
| [backlog.md](backlog.md) | Deferred and future-candidate work |
