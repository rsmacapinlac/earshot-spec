# 0004 — systemd for Service Management

**Status:** Accepted

## Context
The application must start on boot, restart on failure, and be manageable on a
headless Pi with no user session.

## Decision
Run Earshot as a systemd service, installed and enabled by the one-line installer.

## Consequences
- Starts automatically on boot, no user interaction.
- Restarts on crash (`Restart=on-failure`).
- Logs via `journalctl -u earshot`.
- Standard `systemctl start/stop/restart/status earshot`.
- No additional process supervisor required.

> **Implementation note:** the unit runs as the non-root login user selected by
> the installer (`User=<install_user>`, `Group=audio`), with
> `SupplementaryGroups=gpio spi i2c audio` for hardware access and
> `AmbientCapabilities=CAP_SYS_BOOT` (poweroff via `reboot(2)`). The full unit
> contract is in
> [specs/install-service.md](../specs/install-service.md#systemd-service-contract).
