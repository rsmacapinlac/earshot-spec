# 0002 — Python venv over Docker

**Status:** Accepted

## Context
Dependency isolation is needed for the Python application. Docker was considered
as an alternative to a Python virtual environment.

## Decision
Use a Python virtual environment (venv), not Docker. Minimum Python is **3.11**;
the target Raspberry Pi OS ships a newer interpreter (Python 3.13 on Debian 13
"trixie"), so there is no extra Python install step.

## Consequences
- No Docker daemon overhead (~50–100 MB RAM saved, meaningful on the 2 GB target).
- Direct access to GPIO, SPI, and ALSA without `--privileged` or device mounts.
- The seeed-voicecard kernel driver must be installed on the host OS regardless —
  Docker provides no benefit there.
- Simpler systemd config — the service runs the venv Python binary directly (see
  [systemd for service management](0004-systemd-for-service-management.md)).
- For a single-purpose device, Docker's isolation benefits don't outweigh the
  added complexity.
- Docker remains a good fit for a future companion server/API — a separate concern.

> **Implementation note:** the venv lives at `<install_dir>/.venv` (default
> `/home/<install_user>/earshot/.venv`); the service runs
> `<install_dir>/.venv/bin/python -m earshot`. The installer renders these paths
> for the non-root login user running the install.
