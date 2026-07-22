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
- Determine the install identity from the non-root login user running the install
  (`$SUDO_USER` when invoked through `sudo`, otherwise `$USER`), then derive that
  user's home directory, UID, and `<install_dir>` (default `~/earshot`).
- Prompt for the HAT and write `hardware.hat = "respeaker"` to `config.toml`.
- `apt update` and `apt upgrade`.
- Install the ReSpeaker (seeed-voicecard) audio driver.
- Apply the WM8960 capture front-end (ALC speech preset, see
  [recording.md](recording.md#capture-front-end-wm8960-alc)) and persist it to
  `/etc/voicecard/wm8960_asound.state` so it survives reboot.
- Install system audio/media deps: `ffmpeg` (used to encode `session.m4a`).
- Install faster-whisper and pre-download the default transcription model
  (`--no-transcription` skips this; see [processing.md](processing.md#fr-18-installer)).
- Optionally prompt for a **processing service URL** and write `processing.service_url`.
  Leaving it blank is the normal case: the device transcribes locally. It can be set later
  from the web UI (FR-30).

- Create a Python venv (3.11+; uses the OS default interpreter) and install all
  Python dependencies.
- Install and enable the systemd service so Earshot starts on boot.

> A fresh install transcribes on its own — no service, no key, no internet
> ([ADR-0010](../adr/0010-optional-processing-service.md)). A
> [processing service](../../service/README.md) is an optional upgrade that speeds
> transcription up and adds diarization.

**A reboot at the end is required** — the seeed-voicecard driver does not appear
in ALSA until after reboot.

Post-install checks:
```bash
sudo systemctl status earshot
journalctl -u earshot -f
arecord -l          # expect card 'seeed2micvoicec'
```
Then browse to `http://<pi-ip>:<port>/` for the web UI (default port per
[configuration.md](configuration.md#web)). Where the OS mDNS responder is running —
Avahi, present in a default Raspberry Pi OS install — `http://<hostname>.local:<port>/`
also resolves, using the hostname set at flash time. The IP form is the one that always
works; earshot neither installs nor configures mDNS.

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
| `After` / `Wants` | `sound.target network.target` / `sound.target` (`network.target` is ordering only — no start-time network dependency, see [NFR-1](../requirements/non-functional.md#nfr-1-standalone-and-no-internet-dependency)) |
| `Type` | `simple` |
| `User` / `Group` | `<install_user>` / `audio` |
| `WorkingDirectory` | `<install_dir>` (default `/home/<install_user>/earshot`) |
| `ExecStart` | `<install_dir>/.venv/bin/python -m earshot` |
| `ExecReload` | _none_; use `systemctl restart earshot` to apply `config.toml` changes |
| `Restart` / `RestartSec` | `on-failure` / `10` |
| `TimeoutStartSec` | `90` |
| `SupplementaryGroups` | `gpio spi i2c audio` |
| `AmbientCapabilities` | `CAP_SYS_BOOT` |
| Hardening | `NoNewPrivileges=true`, `PrivateTmp=true`, `ProtectSystem=full`, `ReadWritePaths=<install_dir>` |
| Environment | `PYTHONUNBUFFERED=1`; set `XDG_RUNTIME_DIR=/run/user/<install_uid>` only if required by the selected GPIO/SPI/audio libraries |
| `WantedBy` | `multi-user.target` |

`<install_user>`, `<install_uid>`, and `<install_dir>` are installer-rendered values;
the unit must not hardcode a local development username or UID.

Capability rationale:
- `CAP_SYS_BOOT` — safe shutdown via `reboot(2)` (FR-4).
- Supplementary groups — access to the button (`gpio`), APA102 LEDs (`spi`), the
  codec control bus (`i2c`), and ALSA capture (`audio`).

> **Implementation note:** the seeed-voicecard driver is a DKMS out-of-tree
> module; ALSA config for the HAT lives under `/etc/voicecard/`, and the boot overlay
> `dtoverlay=seeed-2mic-voicecard` (plus `dtparam=i2s=on`, SPI) is added to
> `/boot/firmware/config.txt`. See [../reference/respeaker-2mic-hat.md](../reference/respeaker-2mic-hat.md).
