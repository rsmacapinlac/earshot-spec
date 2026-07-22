# Deployment

Container contract and configuration (SR-10). The service ships as a Docker image and is
run with Compose.

## `docker compose up -d`

```yaml
services:
  earshot-processing:
    image: ghcr.io/rsmacapinlac/earshot-processing:latest
    restart: unless-stopped
    ports:
      - "9000:9000"
    volumes:
      - models:/models          # model weights — persistent, or every restart re-downloads
      - state:/state            # job queue + results (SR-9)
    environment:
      EARSHOT_DEVICE: cpu
      EARSHOT_WHISPER_MODEL: base.en
      EARSHOT_MAX_CONCURRENT_JOBS: "1"
      EARSHOT_RESULT_TTL_HOURS: "24"
      HUGGINGFACE_TOKEN: ${HUGGINGFACE_TOKEN}   # first run only, to fetch gated pyannote weights

volumes:
  models:
  state:
```

Both volumes are required. `models` holds multi-gigabyte weights that must survive
restarts (SNFR-1 — nothing is fetched at processing time). `state` holds the job queue and
results so accepted work survives a restart (SR-9). Neither ever holds submitted audio,
which is deleted at job end (SR-8).

## Configuration

All configuration is environment variables; there is no config file.

| Variable | Default | Purpose |
|---|---|---|
| `EARSHOT_PORT` | `9000` | Listen port |
| `EARSHOT_DEVICE` | `cpu` | `cpu` or `cuda` |
| `EARSHOT_WHISPER_MODEL` | `base.en` | faster-whisper model |
| `EARSHOT_DIARIZE_MODEL` | `pyannote/speaker-diarization-3.1` | Diarization pipeline |
| `EARSHOT_MAX_CONCURRENT_JOBS` | `1` | Bounded concurrency (SNFR-5) |
| `EARSHOT_MAX_UPLOAD_MB` | `500` | Upload cap; a 43-min earshot session is ~10 MB |
| `EARSHOT_RESULT_TTL_HOURS` | `24` | Result retention before reaping |
| `HUGGINGFACE_TOKEN` | — | Needed once to fetch gated pyannote weights |

## Host requirements

| | Minimum | Comfortable |
|---|---|---|
| CPU | 4 cores x86-64 or arm64 | 8 cores |
| RAM | 8 GB | 16 GB |
| Disk | 10 GB for weights | — |
| GPU | none | any CUDA card — dramatically faster |

A Raspberry Pi is **not** a supported host. That is the entire point of the split: this
service exists because the recorder cannot do this work
([ADR-0001](../adr/0001-separate-processing-service.md)).

## Network

- One inbound port. No outbound access required once weights are cached (SNFR-1).
- **No authentication in v1** — trusted LAN, matching the device tracks. Anything that can
  reach the port can submit jobs and read results.
- Deploy on a private network. Do not expose the port to the internet: it accepts
  unauthenticated uploads and returns transcripts of private conversations.

## Verifying

```sh
curl -s http://<host>:9000/v1/health | jq
```

Expect `status: "ok"` and both capabilities `true`. If `diarize` is `false`, `detail`
gives the reason — most often the gated pyannote weights were never fetched, which needs
`HUGGINGFACE_TOKEN` set on first run.

Then point a device at `http://<host>:9000` — for the Pi recorder,
`processing.service_url` in its `config.toml`
([rpi configuration](../../rpi/specs/configuration.md#processing)).

## Upgrades

`docker compose pull && docker compose up -d`. Weights persist on the `models` volume;
in-flight jobs are marked `interrupted` per SR-9 and are resubmitted by the caller.
