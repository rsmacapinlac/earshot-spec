# Connectivity

## FR-9: WiFi setup
WiFi configuration is fully delegated to the OS (NetworkManager on Raspberry Pi
OS). The application never manages network connectivity.

### FR-9.1: Primary network
- Configured at flash time via rpi-imager.
- rpi-imager writes credentials to
  `/etc/NetworkManager/system-connections/preconfigured.nmconnection`.
- The device connects automatically on boot when the network is in range.

### FR-9.2: Secondary network (phone hotspot)
- A phone hotspot can be added as a second NetworkManager profile over SSH while
  connected to the primary network.
- Both profiles coexist as independent files under
  `/etc/NetworkManager/system-connections/`; the primary profile is never modified.
- NetworkManager connects automatically to whichever configured network is in range.
- Setup procedure: [../specs/install-service.md](../specs/install-service.md#fr-10-phone-hotspot-setup-optional).

## Application network surface (v1)
The application does not *manage* connectivity (FR-9), but it is no longer network-free.
All of it is **LAN-only — nothing requires the internet** (NFR-1):
- **Inbound:** a web UI over the LAN ([web-ui.md](web-ui.md)), bound to the Pi's IP on a
  configured port. Trusted-LAN, no auth in v1.
- **Outbound:** HTTP to an optional [processing service](../../service/README.md) at
  `processing.service_url`. Unset or unreachable simply means transcription runs locally
  and diarization is not offered; everything else is unaffected.

## Out of scope (v1)
- USB / Bluetooth tethering
- USB cellular modem / LTE dongle
- Captive portal or WiFi onboarding UI
- Adding WiFi networks without SSH access
