# earshot processing service

A containerised service that turns recorded audio into transcripts. It is **not a
device** — it is the compute the earshot recorders don't have.

**It is optional.** The recorders work without it: a Pi transcribes on its own, with no
service, account, or internet. Running this service makes transcription far faster and
adds speaker diarization, which is the one thing a recorder genuinely cannot do. Nothing
breaks if you never deploy it — see
[the recorder's optional-processing-service decision](../rpi/adr/optional-processing-service.md).

## Why it exists

The [Raspberry Pi recorder](../rpi/README.md) is a Pi 4B: excellent at capturing audio
continuously and cheaply, poor at running speech models. It *can* transcribe — 7–13
minutes per 15 minutes of audio, contending with capture for CPU — but speaker
diarization of the quality worth having will not fit at all.

This service is where that work goes **if you have somewhere to put it**. On a capable
host, transcription takes a fraction of the time and diarization becomes possible. Because
it runs on your LAN, the product stays local-first either way: **earshot needs no internet
at all**, with or without this service.

## What it does

1. Accepts an audio file and a job type over HTTP.
2. Runs it — **transcribe** (faster-whisper) or **diarize** (transcribe plus speaker
   labels, via pyannote).
3. Returns structured segments. Rendering them into a transcript is the device's job.
4. Deletes the audio when the job finishes.

Jobs are **asynchronous**: submit, poll, fetch. A long recording takes minutes, so no
request is ever held open for the duration.

## Deployment

`docker compose up -d`, one volume for model weights, one port. See
[`specs/deployment.md`](specs/deployment.md).

Hardware is the operator's choice — an always-on homelab box, a NAS, a spare desktop. A
GPU is optional and dramatically faster; CPU-only works.

## Documentation map

| Area | Directory | What lives here |
|---|---|---|
| Product needs & scope | [`requirements/`](requirements/README.md) | What the service must do, non-functional targets, out of scope |
| Technical decisions | [`adr/`](adr/README.md) | Why the major approaches were chosen |
| Exact behavior (normative) | [`specs/`](specs/README.md) | HTTP API contract, processing pipeline, deployment |

## Relationship to the device tracks

| Track | Uses this service for |
|---|---|
| [`rpi/`](../rpi/README.md) | Faster transcription, and diarization — the only way to get speaker labels. With no service configured the Pi transcribes locally and offers no diarization. |
| [`esp32/`](../esp32/requirements/README.md) | Nothing today. Its v1 transfer seam is a designed-for path to the same API later. |

The service knows nothing about devices — it takes audio and returns segments. That is
what lets one deployment serve both tracks.
