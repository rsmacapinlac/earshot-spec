# Hardware

## Single-board computer

The supported (and minimum) SBC is the **Raspberry Pi 4 Model B**.

| | Pi 4B |
|---|---|
| Model | Raspberry Pi 4 Model B |
| RAM | 2 GB min, 4 GB recommended (8 GB also supported) |
| CPU | Cortex-A72 quad-core |
| OS | Raspberry Pi OS Lite 64-bit |

> The **Pi 4B is the minimum**; smaller boards (Pi Zero 2W, Pi 3B/3B+) are **out of
> scope**. Pi 5 is outside v1 validation scope; it may work, but it is unsupported.
>
> **Target OS:** Raspberry Pi OS (Debian 13 "trixie"), kernel 6.12.x.

## Audio HAT

Earshot uses the **Seeed ReSpeaker 2-Mic Pi HAT** (selected via `hardware.hat`
in `config.toml`). It is the device's entire front panel: mics = record source,
button = control, LEDs = status.

| Component | Detail |
|---|---|
| Audio codec | WM8960 (I²S audio, I²C control) |
| ALSA card name | `seeed-2mic-voicecard` (card id `seeed2micvoicec`) |
| Driver | out-of-tree `snd_soc_seeed_voicecard` (DKMS) |
| Microphones | 2× onboard analog MEMS mics (LINPUT1 / RINPUT1) |
| LEDs | 3× APA102 RGB over SPI (`spidev0.0`) — 1 used in v1 |
| Button | 1 tactile switch on GPIO17 (active-low) |

Full electrical/driver facts and the expected mixer state are in
[../reference/respeaker-2mic-hat.md](../reference/respeaker-2mic-hat.md).
