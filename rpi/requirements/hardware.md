# Hardware

## Single-board computer

The as-built device is a **Raspberry Pi 4 Model B**.

| | Pi 4B (as-built) | Pi Zero 2W (design intent) |
|---|---|---|
| Model | Raspberry Pi 4B (2 GB min, 4 GB recommended) | Raspberry Pi Zero 2W |
| RAM | 4 GB (8 GB on `pi-earshot-pi4`) | 512 MB |
| CPU | Cortex-A72 quad-core | Cortex-A53 quad-core |
| OS | Raspberry Pi OS Lite 64-bit | Raspberry Pi OS Lite 64-bit |
| USB offload | USB-A stick ([FR-11](../specs/usb-offload.md)) | USB OTG gadget mode |
| Typical use | Office/home, desk-mounted | On the go, pocket-sized |

> Pi 3B/3B+ (1 GB) are not supported. Pi 5 is untested.
>
> **As-built:** the running host is `pi-earshot-pi4` on Raspberry Pi OS
> (Debian 13 "trixie"), kernel 6.12.x. The Pi Zero 2W + USB-gadget-mode offload
> path exists in the design but is not the running configuration; it is not
> specified normatively in `specs/`.

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
| Speaker | None (audio output deferred to v2) |

Full electrical/driver facts and the observed mixer state are in
[../reference/respeaker-2mic-hat.md](../reference/respeaker-2mic-hat.md).
