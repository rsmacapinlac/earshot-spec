# Install & Service

Installer contract (FR-8, FR-10) and the systemd unit that runs Earshot on boot
(see [systemd for service management](../adr/0004-systemd-for-service-management.md)).

## FR-8: One-line install
Full setup on a fresh Raspberry Pi OS Lite install, run as the normal login user
(the script uses `sudo` for privileged steps):
```bash
git clone https://github.com/rsmacapinlac/earshot.git ~/earshot
bash ~/earshot/installer/install.sh
```
The installer must:
- Prompt for the HAT and write `hardware.hat = "respeaker"` to `config.toml`.
- `apt update` and `apt upgrade`.
- Install the ReSpeaker (seeed-voicecard) audio driver.
- Apply the WM8960 capture front-end (ALC speech preset, see
  [recording.md](recording.md#capture-front-end-wm8960-alc)) and persist it to
  `/etc/voicecard/wm8960_asound.state` so it survives reboot.
- Install system audio/media deps: `ffmpeg`, `dosfstools`, `mtools`.
- Install faster-whisper and pre-download the default transcription model
  (`--no-transcription` skips this; see [transcription.md](transcription.md#fr-18-installer)).
- Create a Python 3.11 venv and install all Python dependencies.
- Install and enable the systemd service so Earshot starts on boot.

**A reboot at the end is required** — the seeed-voicecard driver does not appear
in ALSA until after reboot.

Post-install checks:
```bash
sudo systemctl status earshot
journalctl -u earshot -f
arecord -l          # expect card 'seeed2micvoicec'
```
Updates: `cd ~/earshot && git pull && bash installer/install.sh`.

## FR-10: Phone hotspot setup (optional)
For portable use, add a phone hotspot as a second WiFi profile over SSH while on
the primary network:
```bash
sudo nmcli connection add type wifi con-name "phone-hotspot" ssid "HotspotSSID" \
  wifi-sec.key-mgmt wpa-psk wifi-sec.psk "YourPassword" \
  connection.autoconnect yes
```
- The hotspot need not be active when the command runs — the profile is saved and
  used automatically when in range.
- Creates a new NetworkManager profile alongside the existing one; the primary
  profile is untouched. No reboot required.
- `nmcli` sets the required `root:root` / mode `600` permissions; NetworkManager
  ignores connection files with wrong permissions.

## systemd service contract
The `earshot.service` unit:

| Field | Value |
|---|---|
| `Description` | Earshot — on-device conversation recorder and transcriber |
| `After` / `Wants` | `sound.target network.target` / `sound.target` |
| `Type` | `simple` |
| `User` / `Group` | `ritchie` / `audio` |
| `WorkingDirectory` | `/home/ritchie/earshot` |
| `ExecStart` | `/home/ritchie/earshot/.venv/bin/python -m earshot` |
| `ExecReload` | `/bin/kill -HUP $MAINPID` |
| `Restart` / `RestartSec` | `on-failure` / `10` |
| `TimeoutStartSec` | `90` |
| `SupplementaryGroups` | `gpio spi i2c audio` |
| `AmbientCapabilities` | `CAP_SYS_BOOT` |
| Hardening | `NoNewPrivileges=true`, `PrivateTmp=true`, `ProtectSystem=full`, `ReadWritePaths=/home/ritchie/earshot` |
| Environment | `PYTHONUNBUFFERED=1`, `XDG_RUNTIME_DIR=/run/user/1000` |
| `WantedBy` | `multi-user.target` |

Capability rationale:
- `CAP_SYS_BOOT` — safe shutdown via `reboot(2)` (FR-4).
- `CAP_SYS_MODULE` / `CAP_SYS_ADMIN` — **not granted**; they are not required for
  Pi 4B USB-A offload.
- Supplementary groups — access to the button (`gpio`), APA102 LEDs (`spi`), the
  codec control bus (`i2c`), and ALSA capture (`audio`).

> **Implementation note:** the seeed-voicecard driver is a DKMS out-of-tree
> module; ALSA config for the HAT lives under `/etc/voicecard/`, and the boot overlay
> `dtoverlay=seeed-2mic-voicecard` (plus `dtparam=i2s=on`, SPI) is added to
> `/boot/firmware/config.txt`. See [../reference/respeaker-2mic-hat.md](../reference/respeaker-2mic-hat.md).
