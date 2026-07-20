# Reference — Seeed ReSpeaker 2-Mic Pi HAT

Non-normative hardware facts and the **observed** configuration on
`pi-earshot-pi4` (captured 2026-07-19). This is the record behind
[requirements/hardware.md](../requirements/hardware.md) and the capture spec in
[specs/recording.md](../specs/recording.md).

## Overview
The ReSpeaker 2-Mic HAT is Earshot's entire front panel: it provides the record
source (2 mics), the sole control (1 button), and the sole status channel (RGB
LEDs). Three functional blocks, all in use:

| Block | Hardware | Wiring | Driver |
|---|---|---|---|
| Audio codec | **WM8960** stereo codec | I²S data + I²C control | out-of-tree `snd_soc_seeed_voicecard` (DKMS) |
| Microphones | 2× analog MEMS → `LINPUT1` / `RINPUT1` | into WM8960 boost mixers | same |
| User button | 1 tactile switch on **GPIO17**, active-low | libgpiod | — |
| RGB LEDs | 3× **APA102** chain over **SPI** | `spidev0.0` / `spidev0.1` | userspace (apa102-pi) |

## ALSA / driver
- Card: **`seeed-2mic-voicecard`**, id `seeed2micvoicec` (card index varies; was
  card 1 on the observed host).
- Codec node: `bcm2835-i2s-wm8960-hifi wm8960-hifi-0`.
- Capture PCM used by Earshot: `plughw:CARD=seeed2micvoicec,DEV=0` (via `arecord`;
  `plughw` handles rate/format conversion).
- Vendor ALSA config under `/etc/voicecard/` defines `dmix`/`dsnoop` array PCMs,
  but Earshot bypasses `!default` and opens `plughw` directly (single consumer,
  no shared mic access).
- Kernel binds the module at boot: `snd_soc_seeed_voicecard` (taints kernel —
  out-of-tree, expected).

## Boot configuration (`/boot/firmware/config.txt`)
Relevant lines observed:
```
dtparam=i2c_arm=on
dtparam=i2s=on
dtoverlay=i2s-mmap
dtoverlay=seeed-2mic-voicecard
enable_uart=1
```

## Capture front-end tuning (observed WM8960 mixer state)
| Control | Value | Meaning |
|---|---|---|
| `Capture` (analog PGA) | 39/63 → **+12 dB**, both channels on | Fixed input gain |
| `ADC PCM` (digital) | 195/255 → **0 dB** | No digital trim |
| `Left/Right Input Mixer Boost` | on | Mic path enabled |
| `Left Boost Mixer LINPUT1` / `Right Boost Mixer RINPUT1` | on | Correct mic inputs selected |
| `ALC Function` | **Off** | No automatic level control — gain is fixed |
| `Noise Gate` | **Off** | Room noise captured verbatim |
| `ADC High Pass Filter` | stock | — |

Implications (see [TD-1](../requirements/open-technical-decisions.md#td-1--capture-gain-fixed-pga-vs-alc)):
- Capture level is a **fixed +12 dB** PGA. With ALC off, distant/quiet speech is
  not boosted and close/loud speech can clip. The WM8960's ALC is available and
  unused — a likely tuning lever for transcription accuracy.
- Noise gate off means constant room noise is present (usually fine for Whisper).
- Both mics are captured as true stereo and stored stereo in `session.wav`; for
  speech-to-text this is largely redundant (see [TD-2](../requirements/open-technical-decisions.md#td-2--stored-wav-stereo-vs-mono)).

## Peripherals summary
| Peripheral | Address / line | Notes |
|---|---|---|
| Button | GPIO17, active-low | `PiButton`; reads HIGH when idle/released |
| LEDs | APA102 ×3 on SPI `spidev0.0` | 1 LED used in v1; full RGB + patterns |
| Speaker | none | HAT has no speaker; audio output deferred to v2 |

## Not part of the ReSpeaker
The observed host also has a **Jabra SPEAK 510** USB speakerphone attached (ALSA
USB-Audio card). It is **not** used by Earshot's capture path and is irrelevant to
this HAT — noted only to avoid confusion when listing `arecord -l`.
