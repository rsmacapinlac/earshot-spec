# 0006 — Filesystem as State, No SQLite

**Status:** Accepted

## Context
SQLite was previously chosen to track recording lifecycle and API upload state.
With API sync removed, upload
tracking disappears. The remaining state — whether a chunk is encoded, whether
encoding failed, whether a session is transcribed — is representable directly by
files in each session directory.

## Decision
Use the filesystem as the sole state store. No SQLite. State is derived from the
presence/combination of files in each session directory. Application errors are
logged to the systemd journal (see
[systemd for service management](0004-systemd-for-service-management.md)).

See [specs/storage.md](../specs/storage.md#filesystem-as-state) for the
authoritative state table.

## Consequences
- No SQLite dependency, no schema migrations, no database file.
- Recovery on boot is a filesystem scan, not a query — equally simple at this scale.
- No cross-recording query capability (only used for upload tracking, now gone).

> **Implementation note:** a small `status.json` per session (written for the
> companion `earshot-tui`) mirrors the derived state (`encoded` → `transcribed`)
> but is **not** the source of truth — file presence is. See
> [specs/storage.md](../specs/storage.md).
