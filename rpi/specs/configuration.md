# Configuration

Earshot reads a single `config.toml` at `~/earshot/config.toml`. The installer
creates it interactively. All keys have defaults; omitting a key uses the default.
Apply changes with `sudo systemctl restart earshot`.

> **Authoritative schema.** The keys below define the config the application
> parses. Earlier drafts used `[encoding]`/`[shutdown]`/`storage.recordings_dir`, and a
> `[diarization]` section for an OpenAI key; both are superseded — the keys below win.

## `[hardware]`
| Key | Type | Default | Description |
|---|---|---|---|
| `hat` | string | `"respeaker"` | Audio HAT, selected at startup by the HAL ([ADR-0003](../adr/0003-hardware-abstraction-layer.md)). Written by the installer. |

## `[audio]`
| Key | Type | Default | Description |
|---|---|---|---|
| `sample_rate` | int | `16000` | Capture sample rate (Hz). |
| `channels` | int | `1` | Capture channels — mono, the left ReSpeaker mic (see [recording.md](recording.md#capture-spec)). |
| `bit_depth` | int | `16` | PCM bit depth. |
| `alsa_pcm` | string | `"plughw:CARD=seeed2micvoicec,DEV=0"` | ALSA capture PCM for `arecord`. Use `plughw:` for rate/format conversion. |

## `[recording]`
| Key | Type | Default | Description |
|---|---|---|---|
| `chunk_duration_seconds` | int | `900` | Length of each WAV chunk (15 min). Recording continues seamlessly across chunks. |
| `min_duration_seconds` | int | `3` | Sessions shorter than this are discarded. |
| `encode_bitrate_kbps` | int | `32` | AAC bitrate for `session.m4a`. 32 kbps suits 16 kHz mono speech (~0.24 MB/min); raise to 64 for more headroom. **Provisional — confirm on hardware during bring-up** ([recording.md](recording.md#size-reference)); the encode is one-way. Container and codec are fixed — see [ADR-0001](../adr/0001-audio-storage-format.md). |
| `shutdown_hold_seconds` | int | `3` | Button hold (while idle) that triggers safe shutdown. |

## `[storage]`
| Key | Type | Default | Description |
|---|---|---|---|
| `data_dir` | string | `"~/earshot"` | Base data directory. Recordings are written under `<data_dir>/recordings/`. |
| `disk_threshold_percent` | int | `90` | Disk usage at which new recordings are blocked. |

## `[transcription]`
The **local** transcription engine — the default path, used whenever no processing service
is configured. The device transcribes on its own with no service, key, or internet
([ADR-0010](../adr/0010-optional-processing-service.md)).

| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `true` | Enable local transcription. `false` means transcripts require a configured service. |
| `model` | string | `"base.en"` | `"base.en"` (INT8, ~60 MB) — more accurate on the Pi 4B. `"tiny.en"` (INT8, ~35 MB) is faster and lighter. |
| `threads` | int | `2` | faster-whisper `cpu_threads`. Default 2 leaves headroom for recording on the 4-core CPU. |

## `[processing]`
An **optional** [processing service](../../service/README.md) on the LAN. Setting a URL
routes transcription there instead of running it locally, and is the only way to enable
diarization. Leaving it empty is a fully supported configuration.

| Key | Type | Default | Description |
|---|---|---|---|
| `service_url` | string | `""` | Base URL, e.g. `"http://homelab.local:9000"`. Empty means transcription runs locally and **diarization is unavailable** (FR-25 is not offered). Settable from the web UI (FR-30). |
| `poll_interval_seconds` | int | `5` | How often an in-flight service job is polled. Unused when no service is set. |
| `max_failures` | int | `3` | Failed processing attempts per session before writing `.failed_processing` and skipping it. `0` retries forever. An unreachable service does **not** count as a session failure. |

## `[web]`
| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `true` | Serve the web UI ([web-ui.md](../requirements/web-ui.md)). |
| `bind_address` | string | `"0.0.0.0"` | Interface to bind. Default binds all interfaces so the UI is reachable at the Pi's LAN IP (trusted-LAN, no auth — v1). |
| `port` | int | `8080` | TCP port for the web UI. |

## Example `config.toml`
```toml
[hardware]
hat = "respeaker"

[audio]
sample_rate = 16000
channels = 1
bit_depth = 16
alsa_pcm = "plughw:CARD=seeed2micvoicec,DEV=0"

[recording]
chunk_duration_seconds = 900
min_duration_seconds = 3
encode_bitrate_kbps = 32
shutdown_hold_seconds = 3

[storage]
data_dir = "~/earshot"
disk_threshold_percent = 90

[transcription]
enabled = true
model = "base.en"
threads = 2

[processing]
service_url = ""
poll_interval_seconds = 5
max_failures = 3

[web]
enabled = true
bind_address = "0.0.0.0"
port = 8080
```
