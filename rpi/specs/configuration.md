# Configuration

Earshot reads a single `config.toml` at `~/earshot/config.toml`. The installer
creates it interactively. All keys have defaults; omitting a key uses the default.
Apply changes with `sudo systemctl restart earshot`.

> **Authoritative schema.** The keys below define the config the application
> parses. An earlier draft used a different schema (`[encoding]`, `[shutdown]`,
> `storage.recordings_dir`); it is superseded — the keys below win.

## `[hardware]`
| Key | Type | Default | Values | Description |
|---|---|---|---|---|
| `hat` | string | `"respeaker"` | `"respeaker"` | Audio HAT to use. |

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
| `shutdown_hold_seconds` | int | `3` | Button hold (while idle) that triggers safe shutdown. |

## `[storage]`
| Key | Type | Default | Description |
|---|---|---|---|
| `data_dir` | string | `"~/earshot"` | Base data directory. Recordings are written under `<data_dir>/recordings/`. |
| `disk_threshold_percent` | int | `90` | Disk usage at which new recordings are blocked. |

## `[display]`
| Key | Type | Default | Description |
|---|---|---|---|
| `brightness` | int | `80` | Display brightness (0–100). No-op on the ReSpeaker HAT (no display). |

## `[transcription]`
| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `true` | Enable on-device transcription. `false` disables it (no amber state, no `transcript.md`). |
| `model` | string | `"base.en"` | `"base.en"` (INT8, ~60 MB) default — more accurate on the Pi 4B. `"tiny.en"` (INT8, ~35 MB) is a faster, lighter alternative. |
| `threads` | int | `2` | faster-whisper `cpu_threads`. Default 2 leaves headroom for recording on the 4-core CPU. |

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
shutdown_hold_seconds = 3

[storage]
data_dir = "~/earshot"
disk_threshold_percent = 90

[display]
brightness = 80

[transcription]
enabled = true
model = "base.en"
threads = 2
```
