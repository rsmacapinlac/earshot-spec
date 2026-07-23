# Sessions are Numbered (and use the database's primary key)

**Status:** Accepted

## Context

A session needs a name — the queue orders by it, crash recovery scans for it, transcripts
cite it.

## Decision

Every recording gets a number, from the database.

State already lives in SQLite ([state storage](state-storage.md)), so the number is that
store's own key: starting a recording inserts a `sessions` row, and its
`INTEGER PRIMARY KEY AUTOINCREMENT` *is* the session's identity.

The session directory is named by rendering that integer as `rec-` plus six zero-padded
digits — `42` → `rec-000042`. Padding is presentation only, so it can widen past six
digits without breaking anything.

**`AUTOINCREMENT` is required, not incidental.** A plain `INTEGER PRIMARY KEY` is a rowid
alias, and SQLite reuses rowids once the highest row is deleted. The keyword backs the
column with `sqlite_sequence` and guarantees monotonicity. Contract:
[storage.md](../specs/storage.md#session-identity).

**This puts the database on the capture path.** Starting a recording writes a row before
any audio exists. Accepted deliberately: `earshot.db` sits on the same filesystem as the
audio, so there is no failure mode where the database is unwritable but the WAV chunk
writer is not. "Capture depends on nothing external" concerns services, networks, and
clocks — a local file is none of those. See
[storage.md](../specs/storage.md#the-database-is-on-the-capture-path).

Capture time demotes to metadata: the session row carries `created_at`, set at insert, and
`status.json` mirrors it. Nothing sorts, looks up, or recovers by time.

This matches the ESP32 track, which reached the same conclusion for the same reason —
stable ID as identity, optional human label on top.

## Consequences

- **Queue order is capture order.** IDs are monotonic, so processing oldest-first is a
  plain sort by ID — no clock consulted ([clock independence](../requirements/non-functional/clock-independence.md)).
- **IDs are never reused.** A reference to `rec-000042` — a downloaded transcript, a
  bookmarked URL — always resolves to the same recording, even after it is deleted.
  `AUTOINCREMENT` guarantees this.
- **Allocation cannot race.** The button and the web UI can both start a recording; the
  database serialises the insert, so identity does not depend on the state machine
  enforcing one-at-a-time.
- **The capture time is recorded at insert**, as `created_at`, rather than held in memory
  until finalization — which also removes the crash case where an interrupted session lost
  its start time.
- **The directory name is a public identifier.** It appears in URLs and in a downloaded
  transcript's `**Session:**` line, so anything referencing a recording depends on the
  `rec-NNNNNN` form being stable.

## Alternatives

- **A filesystem scan — `max + 1` over existing `rec-*` directories** — no database at the
  moment of capture. Rejected: deleting the newest session frees its ID for reuse, and a
  read-modify-write is correct only while exactly one caller allocates at a time.
- **A UUID** — globally unique with no coordination. Rejected: it carries no order (UUIDv7
  sorts only by embedding a timestamp), so the queue would need another sort key anyway,
  and opaque identifiers cost legibility for uniqueness nothing here needs. A UUID column
  can be added later for shareable URLs without disturbing this.
