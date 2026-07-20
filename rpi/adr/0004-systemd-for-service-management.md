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

> **As-built:** the unit runs as `User=ritchie`, `Group=audio`, with
> `SupplementaryGroups=gpio spi i2c audio` for hardware access and
> `AmbientCapabilities=CAP_SYS_BOOT …` (poweroff via `reboot(2)`; the unit also
> carries `CAP_SYS_MODULE`/`CAP_SYS_ADMIN`, which are not required for Pi 4B and
> can be dropped). The full unit contract is in
> [specs/install-service.md](../specs/install-service.md#systemd-service-contract).
