# No internet; graceful degradation

## Requirement

Two absolutes:

- **No earshot feature requires the internet.** Not recording, not transcription, not
  diarization. There is no account, no API key, and no third party anywhere in the system.
- **Capture depends on nothing external.** Recording, chunk rollover, finalization and
  storage work with no network at all. Audio is never put at risk by a missing network,
  service, or host.

Everything else degrades by what is available.

## Capability tiers

| Available | What the device can do |
|---|---|
| **No network** | Record, stop and shut down from the button; LED status. Audio is captured, encoded, and waits safely. Nothing else is reachable. |
| **LAN** | Everything except diarization — web UI, browse, play, download, delete, name sessions, device status, and **local transcription**. |
| **LAN + [processing service](../../../service/README.md)** | Faster transcription, and diarization: the one capability the Pi cannot provide itself ([optional processing service](../../adr/optional-processing-service.md)). |
| **Internet** | Nothing. It is never required. |

## The limit worth stating

**Transcription needs a LAN — not because it runs remotely, but because it is
web-initiated.** The engine is on the device, yet every job is started from the web UI
([transcribe](../web-ui/transcribe.md)) and the button has no gesture to spare: both are taken by record/stop and
shutdown-hold. An unnetworked Pi therefore records reliably and transcribes never.

Accepted for a mains-powered desk device expected to be on wifi, and recorded here so the
limit is explicit rather than implied away by "works offline".

## Where this is specified

- [`specs/processing.md`](../../specs/processing.md) — the local and service routes, and
  which needs what.
- [`connectivity.md`](../connectivity.md) — the application's inbound and outbound
  network surface.
- [`adr/optional-processing-service.md`](../../adr/optional-processing-service.md)
  — why the service is an upgrade rather than a dependency.
