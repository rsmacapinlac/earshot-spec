# SQLite for State, Files for Artifacts

**Status:** Accepted · **Re-decided 2026-07-22** — state was previously held as files in
each session directory; see [History](#history).

## Context

State began small. An early design used SQLite to track recording lifecycle and API upload
state; when API sync was removed, what remained was representable as file presence — does
`session.m4a` exist, is there a `transcript.md` — so SQLite was dropped for the filesystem.

That premise has expired. State has grown back, and changed character:

- **Mutable, user-authored data** — session names, `Speaker N` → name maps, rewritten in
  place from the web UI.
- **Transient operational data** — in-flight job identifiers, attempt counts, last errors.
- **A processing queue** — work that is enqueued, ordered, retried, and must survive a
  reboot.
- **Two concurrent writers** — the web UI and the job worker both mutate session state.

Only the first category was ever presence-derived. The rest is *file contents with a
schema*, and by the end a session directory carried five separate state files — one of
which gained a field within a day of being created.

Two gaps followed, neither addressed anywhere:

- **Atomicity.** Nothing specified how state files were written. A power loss mid-rewrite
  truncates the JSON and silently costs a session its name and speaker map.
- **Concurrency.** Two writers, no lock, no defined ordering.

Both are solvable by hand — write-temp-then-rename, an explicit mutex — but that is
discipline enforced by review across a dozen call sites, on a device that runs unattended.

## Decision

**SQLite holds state. The filesystem holds artifacts.**

- **On disk, per session:** `session.m4a`, `transcript.md`, the raw segment JSON, and
  `status.json`. These are the durable outputs — recoverable with nothing but a card
  reader, and never inside a database.
- **In `<data_dir>/earshot.db`:** session records, speaker name maps, the processing
  queue, and job history.

Run in **WAL mode** — one writer, concurrent readers, crash-safe commits. `sqlite3` is in
the Python standard library, so this adds no dependency, no daemon, and no service.

**The database is on the capture path, and that is fine.** A session ID is allocated by
inserting a row (see [session identity](session-identity.md)). `earshot.db` lives on the same
filesystem as the audio, so there is no failure mode where the database is unavailable but
the WAV chunk writer is not — the shared risk is the disk, and it is shared either way.
"Capture depends on nothing external" is about services, networks, and clocks; a local
file is none of those.

**The database is rebuildable.** `status.json` is written per session and carries enough —
name, speakers, timings, state — to reconstruct a row. A lost
database is recovered by scanning `recordings/` and reading those files. Nothing
irreplaceable lives only in the database.

## Consequences

- **Atomicity and concurrency come from the store**, not from discipline. Renaming a
  session and rewriting its transcript header is one transaction.
- **The session directory drops from ten file types to five.** `session.json`,
  `.job.json`, `.processing_failures.json`, `.failed_processing`, and `recordings/.next_id`
  all disappear.
- **A real queue becomes cheap** — ordered, durable, retryable, inspectable — where before
  it was inferred by scanning for `session.m4a` without `transcript.md`. The web UI can
  show what is queued, not only what is running.
- **IDs stop being reusable.** Filesystem allocation was `max + 1` over existing
  directories, so deleting the newest session let the next recording reuse its ID.
  Auto-increment makes that impossible ([session identity](session-identity.md)).
- **Cost: a schema, and migrations.** The genuine price. Schema changes now need a
  versioned migration step rather than tolerating a missing JSON key.
- **Cost: two places to look.** Debugging means reading files *and* querying a database,
  where `ls` used to tell most of the story.
- **`status.json` gains fields and importance.** Still non-authoritative, but it is now the
  rebuild path, so it must be written on every state change rather than only at session
  end.
- **Reconciliation replaces derivation.** Sessions are born in the database; a row with no
  directory is a failed or deleted session, a directory with no row is adopted. See
  [storage.md](../specs/storage.md#reconciliation).

## Alternatives

- **Keep filesystem-as-state, hardened** — write-temp-then-rename everywhere, consolidate
  the state files, add an explicit lock. Rejected: it reimplements a transaction log by
  hand, and correctness then depends on every future call site remembering the discipline.
- **SQLite as a rebuildable index only**, filesystem authoritative — nothing at risk.
  Rejected: every write goes to two places and the two can disagree, which is a worse
  consistency problem than the one being solved.
- **Put audio and transcripts in the database** — one store, no reconciliation. Rejected:
  recordings must stay ordinary files. Recovering audio from a dead device with a card
  reader is worth more than schema tidiness.
- **Keep filesystem ID allocation to keep the DB off the capture path** — rejected once it
  was clear the database is not an external dependency, and that filesystem allocation
  reuses IDs after a delete.

## History

This record previously decided **filesystem-as-state, no SQLite**, on the grounds that
removing API sync had left too little state to justify a database. That was right at the
time. What changed is not the reasoning but the system: the web UI added user-authored
labels and a second writer, and off-device processing added durable job state. SQLite was
dropped when state shrank; it returns now that state has grown, with concurrency it did
not previously have.
