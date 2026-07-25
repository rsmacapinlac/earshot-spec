# Configuration

Earshot reads a single `config.toml` in the **data directory** — `~/earshot-data/config.toml`
by default, not the install directory. The installer creates it interactively. All keys
have defaults; omitting a key uses the default.

**Configuration is done by editing `config.toml` over SSH** and applying with
`sudo systemctl restart earshot`. This is the intended workflow, not a limitation: whoever
owns the device already reached it over SSH to install it, and configuration changes are
rare. The web UI is for operating the device — recording, browsing, transcribing — not for
editing its config. The one exception is `[processing].service_url`, which the UI can set
because managing the service connection is an operational act with live status (see
[configure a processing service](../requirements/web-ui/processing-service.md)); a change
to it applies immediately without a restart.

> **Authoritative schema.** The keys below define the config the application
> parses. Earlier drafts used `[encoding]`/`[shutdown]`/`storage.recordings_dir`, and a
> `[diarization]` section for an OpenAI key; both are superseded — the keys below win.

## `[hardware]`
| Key | Type | Default | Description |
|---|---|---|---|
| `hat` | string | `"respeaker"` | Audio HAT, selected at startup by the HAL ([hardware abstraction layer](../adr/hardware-abstraction-layer.md)). Written by the installer. |

## `[audio]`
| Key | Type | Default | Description |
|---|---|---|---|
| `sample_rate` | int | `16000` | Capture sample rate (Hz). |
| `channels` | int | `1` | Capture channels — mono, the left ReSpeaker mic (see [recording.md](recording.md#capture-spec)). |
| `bit_depth` | int | `16` | PCM bit depth. |
| `alsa_pcm` | string | `"plughw:CARD=seeed2micvoicec,DEV=0"` | ALSA capture PCM for `arecord`. Use `plughw:` for rate/format conversion. |

## `[recording]`
A change takes effect on the **next** recording; it never disturbs one in progress.

| Key | Type | Default | Description |
|---|---|---|---|
| `chunk_duration_seconds` | int | `900` | Length of each WAV chunk (15 min). Recording continues seamlessly across chunks. |
| `min_duration_seconds` | int | `3` | Sessions shorter than this are discarded. |
| `encode_bitrate_kbps` | int | `32` | AAC bitrate for `session.m4a`. 32 kbps suits 16 kHz mono speech (~0.24 MB/min); raise to 64 for more headroom. **Provisional — confirm on hardware during bring-up** ([recording.md](recording.md#size-reference)); the encode is one-way. Container and codec are fixed — see [audio storage format](../adr/audio-storage-format.md). |
| `shutdown_hold_seconds` | int | `3` | Button hold (while idle) that triggers safe shutdown. |

## `[storage]`
| Key | Type | Default | Description |
|---|---|---|---|
| `data_dir` | string | `"~/earshot-data"` | Everything the device owns — `config.toml`, `earshot.db`, and `recordings/`. **Deliberately separate from the install directory** (`~/earshot`), which is a git checkout whose update path is `git pull`. |
| `disk_threshold_percent` | int | `90` | Disk usage at which new recordings are blocked. |

## `[transcription]`
The **local** transcription engine — the default path, used whenever no processing service
is configured. The device transcribes on its own with no service, key, or internet
([optional processing service](../adr/optional-processing-service.md)).

| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `true` | Enable local transcription. `false` means transcripts require a configured service. |
| `model` | string | `"base.en"` | `"base.en"` (INT8, ~60 MB) — more accurate on the Pi 4B. `"tiny.en"` (INT8, ~35 MB) is faster and lighter. |
| `threads` | int | `2` | faster-whisper `cpu_threads`. Default 2 leaves headroom for recording on the 4-core CPU. |

## `[processing]`
An **optional** [processing service](../reference/processing-service.md) on the LAN. Setting a URL
routes transcription there instead of running it locally, and is the only way to enable
diarization. Leaving it empty is a fully supported configuration.

| Key | Type | Default | Description |
|---|---|---|---|
| `service_url` | string | `""` | Base URL, e.g. `"http://homelab.local:9010"`. Empty means transcription runs locally and **[diarization](../requirements/web-ui/diarize.md) is unavailable**. Settable from the [web UI](../requirements/web-ui/processing-service.md). |
| `request_timeout_seconds` | int | `0` | Client timeout for a synchronous `/asr` request. `0` means no timeout — a long session may hold the connection for tens of minutes ([throughput](../reference/processing-service.md#throughput)). A timeout fails the attempt like any other error. |
| `max_failures` | int | `3` | Attempts before a job is marked `failed` and stops being retried. `0` retries forever. An unreachable service does **not** count as an attempt. |

## `[web]`
| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `true` | Serve the [web UI](../requirements/web-ui/README.md). |
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
data_dir = "~/earshot-data"
disk_threshold_percent = 90

[transcription]
enabled = true
model = "base.en"
threads = 2

[processing]
service_url = ""
request_timeout_seconds = 0
max_failures = 3

[web]
enabled = true
bind_address = "0.0.0.0"
port = 8080
```
