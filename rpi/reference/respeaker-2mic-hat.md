# Reference — Seeed ReSpeaker 2-Mic Pi HAT

Non-normative hardware facts and the **expected** device configuration for the
ReSpeaker 2-Mic HAT on Raspberry Pi OS. This is the record behind
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
- Card: **`seeed-2mic-voicecard`**, id `seeed2micvoicec` (card index varies).
- Codec node: `bcm2835-i2s-wm8960-hifi wm8960-hifi-0`.
- Capture PCM used by Earshot: `plughw:CARD=seeed2micvoicec,DEV=0` (via `arecord`;
  `plughw` handles rate/format conversion).
- Vendor ALSA config under `/etc/voicecard/` defines `dmix`/`dsnoop` array PCMs,
  but Earshot bypasses `!default` and opens `plughw` directly (single consumer,
  no shared mic access).
- Kernel binds the module at boot: `snd_soc_seeed_voicecard` (taints kernel —
  out-of-tree, expected).

## Boot configuration (`/boot/firmware/config.txt`)
Relevant lines:
```
dtparam=i2c_arm=on
dtparam=i2s=on
dtparam=spi=on
dtoverlay=i2s-mmap
dtoverlay=seeed-2mic-voicecard
enable_uart=1
```

## Capture front-end

Earshot configures the WM8960 for **ALC (Automatic Level Control)** using the
speech preset — the required configuration is normative in
[specs/recording.md](../specs/recording.md#capture-front-end-wm8960-alc). This
follows Wolfson's and the ReSpeaker community's guidance that voice capture on this
codec should use ALC (fixed gain clips loud speech and under-records quiet speech;
the HAT ships with ALC disabled).

**Shipped default (WM8960 driver default, before tuning):**

| Control | Value | Meaning |
|---|---|---|
| `Capture` (analog PGA) | 39/63 → **+12 dB**, both channels on | Fixed input gain |
| `ADC PCM` (digital) | 195/255 → **0 dB** | No digital trim |
| `Left/Right Input Mixer Boost` | on | Mic path enabled |
| `Left Boost Mixer LINPUT1` / `Right Boost Mixer RINPUT1` | on | Correct mic inputs selected |
| `ALC Function` | **Off** | No automatic level control |
| `Noise Gate` | **Off** | Room noise captured verbatim |
| `ADC High Pass Filter` | stock | — |

The HAT has two mics, but Earshot stores **mono** from the **left** mic only
(faster-whisper downmixes to mono anyway, and the closely-spaced mics carry no
usable stereo image) — see [specs/recording.md](../specs/recording.md#capture-spec).

**Sources:**
[Wolfson WAN0140 — ALC for Portable Applications](https://statics.cirrus.com/pubs/appNote/WAN0140.pdf),
[WM8960 datasheet](https://cdn.sparkfun.com/assets/a/3/a/7/4/WM8960_datasheet_v4.2.pdf),
[Rhasspy — Ideal settings for ReSpeaker 2-Mic HAT](https://community.rhasspy.org/t/ideal-settings-for-respeaker-2-mic-hat/2122),
[Seeed Studio Wiki — ReSpeaker 2-Mics Pi HAT](https://wiki.seeedstudio.com/ReSpeaker_2_Mics_Pi_HAT/).

## Peripherals summary
| Peripheral | Address / line | Notes |
|---|---|---|
| Button | GPIO17, active-low | `PiButton`; reads HIGH when idle/released |
| LEDs | APA102 ×3 on SPI `spidev0.0` | 1 LED used in v1; full RGB + patterns |
