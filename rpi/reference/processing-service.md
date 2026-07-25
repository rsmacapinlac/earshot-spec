# Processing service (reference)

Non-normative operational reference for the optional processing service the Pi can offload
to. **Why** earshot adopts an off-the-shelf service rather than building one:
[adopt an off-the-shelf processing service](../adr/off-the-shelf-processing-service.md).
**Which** service, and its validation:
[experiment 0002](../experiments/0002-whisper-asr-webservice.md). **The contract the device
is built against** (normative): [processing.md](../specs/processing.md#fr-15b-process--service).

A processing service is an **optional upgrade**: it makes transcription faster and is the
only route to [diarization](../requirements/web-ui/diarize.md). The device stands alone
without one ([optional processing service](../adr/optional-processing-service.md)). It is
**not** an earshot-built component — the recorder integrates against a third-party service
running on the operator's LAN.

## Adopted service

`onerahmet/openai-whisper-asr-webservice` (WhisperX) — the Docker Hub image for the
[`ahmetoner/whisper-asr-webservice`](https://github.com/ahmetoner/whisper-asr-webservice)
project:

- **Image / tag:** `onerahmet/openai-whisper-asr-webservice:v1.9.1` — **pin a version; do
  not run `:latest`.** A `:latest` build was observed (2026-07-25) to 500 on every request
  from a WhisperX engine regression (`self.model` `None` in `load_model`), and an auto-updater
  (e.g. `diun`/watchtower) pulling `:latest` can break a working host silently
  ([experiment 0002](../experiments/0002-whisper-asr-webservice.md)).
- **Engine:** `ASR_ENGINE=whisperx` (faster-whisper transcription + pyannote diarization)
- **Interface:** a single **synchronous** `POST /asr`; also `POST /detect-language`.
  Swagger at `/docs`, schema at `/openapi.json`. No jobs, polling, cancel, or health
  endpoint — the device owns those concerns.

## Deployment (`docker compose up -d`)

```yaml
services:
  earshot-processing:
    image: onerahmet/openai-whisper-asr-webservice:v1.9.1   # pin a version; :latest can regress
    restart: unless-stopped
    ports:
      - "9010:9000"                # host 9010 -> container 9000; the device is pointed at host:port
    volumes:
      - cache:/root/.cache         # whisper + pyannote weights — persist or every restart re-downloads
    environment:
      ASR_ENGINE: whisperx
      ASR_MODEL: base
      HF_TOKEN: ${HF_TOKEN}        # required to pull the gated pyannote diarization models

volumes:
  cache:
```

- The `cache` volume holds the downloaded weights and **must persist** across restarts
  (nothing is fetched at processing time). The service is otherwise stateless — no jobs,
  results, or submitted audio persist.
- **Accept the pyannote license once.** The pyannote `speaker-diarization` models are gated
  on HuggingFace: accept their terms on the HF website with the account the `HF_TOKEN`
  belongs to, once, before the first diarization run. Without an accepted license + token,
  `diarize=true` fails while `transcribe` keeps working.
- **Device vs GPU** selection and quantization follow the upstream image's own options and
  image tags; see its documentation.

## Host requirements

| | Minimum | Comfortable |
|---|---|---|
| CPU | 4 cores x86-64 or arm64 | 8 cores |
| RAM | 8 GB | 16 GB — pyannote adds a few GB resident on top of whisper |
| Disk | ~10 GB for weights on the `cache` volume | — |
| GPU | none | any CUDA card — dramatically faster |

A Raspberry Pi is **not** a supported host — the whole point of offloading is that the
recorder cannot do this work.

## Throughput

- **Measured (2026-07-25, this host):** ~94 s for a 90 s clip with `diarize=true` — roughly
  **1× realtime** on CPU. A 43-minute session would take on the order of ~45 minutes.
- Even at ~1× realtime this beats the Pi's local transcription (~7–13 min per 15 min), and
  diarization has **no** local path. But the "much faster" margin is **host-dependent**: a
  GPU host is dramatically quicker, a modest CPU host may sit near 1× realtime. Confirm a
  given host before promising a speedup.

## Network & security

- One inbound port. **No authentication** — trusted LAN, matching the device tracks.
  Anything that can reach the port can submit audio and read transcripts.
- Deploy on a private network. Do **not** expose the port to the internet.
- No outbound access is required once weights are cached, apart from the one-time model
  download on first run.

## Verifying

There is no health endpoint. Confirm two ways:

```sh
# API up + capability discovery (how the device learns `diarize` is offered)
curl -s http://<host>:9010/openapi.json | grep -o '"diarize"'

# a real transcription round-trips
curl -s -X POST \
  "http://<host>:9010/asr?task=transcribe&output=json&encode=true" \
  -F "audio_file=@clip.wav;type=audio/wav" | head
```

Then point the device at `http://<host>:9010` — `processing.service_url` in the Pi's
`config.toml` ([configuration](../specs/configuration.md#processing)), settable from the
[web UI](../requirements/web-ui/processing-service.md).

## Upgrades

`docker compose pull && docker compose up -d`. Weights persist on the `cache` volume.
Because the service is stateless, no in-flight work is lost on the service side; a device
whose request was interrupted mid-upgrade simply re-runs it
([crash resilience](../specs/processing.md#crash-resilience)).
