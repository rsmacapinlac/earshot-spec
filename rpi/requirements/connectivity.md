# Connectivity

WiFi configuration is fully delegated to the OS (NetworkManager on Raspberry Pi
OS). The application never manages network connectivity.

Configured at flash time via rpi-imager.

The application does not *manage* connectivity. Its entire user interface is served over the LAN.

## Out of scope (v1)
- USB / Bluetooth tethering
- USB cellular modem / LTE dongle
- Captive portal or WiFi onboarding UI
- Adding WiFi networks without SSH access
