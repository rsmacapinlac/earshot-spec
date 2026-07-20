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
The application does not *manage* connectivity (FR-9), but it is no longer network-free:
- It serves a **web UI over the LAN** ([web-ui.md](web-ui.md)), bound to the Pi's IP on
  a configured port. This is the app's only inbound network surface (trusted-LAN, no
  auth in v1).
- **Diarization** (FR-25) makes outbound calls to OpenAI and therefore needs internet.
  It is the only feature that does; recording and local transcription stay fully offline.

## Out of scope (v1)
- USB / Bluetooth tethering
- USB cellular modem / LTE dongle
- Captive portal or WiFi onboarding UI
- Adding WiFi networks without SSH access
