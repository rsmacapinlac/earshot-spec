# Storage

Where recordings and state live, how they relate, and how they recover (FR-7).

**Artifacts are files; state is a database** — see
[SQLite for state, files for artifacts](../adr/state-storage.md). The stored audio
artifact is a single m4a (see
[Audio storage format](../adr/audio-storage-format.md)).

## FR-7: Local storage

Everything the device owns lives under `storage.data_dir` (default `~/earshot-data`):

```text
~/earshot-data/
  earshot.db                     state — sessions, speakers, jobs
  config.toml                    configuration
  recordings/
    rec-000042/
      session.m4a                the retained audio
      transcript.md              the transcript, once produced
      transcript_raw.json        raw segments from the last job
      status.json                earshot-tui export, and the DB rebuild path
```

**The data directory is deliberately separate from the install directory.** The installer
clones to `~/earshot`, which holds the checkout, the venv, and `installer/` — and whose
documented update path is `git pull`. No user data belongs in a git working tree.

Recordings stay on the device until deleted. A session directory is removed whole.

### Session identity

A session's identity is an integer allocated by the database, rendered as a directory name
of `rec-` plus six zero-padded digits ([session identity](../adr/session-identity.md)).

- Allocation is the `INTEGER PRIMARY KEY AUTOINCREMENT` of the inserted `sessions` row.
  **The `AUTOINCREMENT` keyword is required** — a plain integer primary key is a rowid
  alias and SQLite reuses rowids after the highest row is deleted, which is precisely the
  behaviour being removed.
- **IDs are never reused.** A reference to `rec-000042` always means the same recording,
  even after it is deleted.
- Allocation consults no clock
  ([clock independence](../requirements/non-functional/clock-independence.md)), so a
  session captured on a device that has never had a valid time is indistinguishable from
  any other.
- Padding is presentation. The database holds `42`; nothing parses the directory name back
  into an integer for identity purposes.

## State — `earshot.db`

SQLite in **WAL mode**: one writer, concurrent readers, crash-safe commits.

| Table | Holds |
|---|---|
| `sessions` | id, name, `created_at`, duration, derived state, whether the transcript is diarized |
| `speakers` | per session, the `Speaker N` → assigned-name map |
| `jobs` | the processing queue and its history — see [processing.md](processing.md#the-queue) |

- **`name` is nullable.** Null means unnamed, and the UI falls back to the rendered ID.
  Names are not unique and nothing looks a session up by name.
- **`created_at` is set at insert**, when the recording starts. It is descriptive metadata:
  nothing sorts, looks up, or recovers by it, and on a device with no RTC it may be wrong.
  Ordering uses the ID.
- **A speaker label with no row stays `Speaker N`.**

### The database is on the capture path

Starting a recording inserts a row before any audio is written. This is accepted:
`earshot.db` sits on the same filesystem as the audio, so there is no failure mode where
the database is unwritable but the WAV chunk writer is not. The principle that capture
depends on nothing external concerns services, networks, and clocks — not local files.

### The database is rebuildable

`status.json` is written per session on every state change, and carries enough to
reconstruct a row:

```json
{ "status": "encoded" | "transcribed" | "diarized",
  "device": "earshot-pi",
  "name": "Weekly sync — pricing",
  "speakers": { "Speaker 1": "Ritchie", "Speaker 2": "Sarah" },
  "created_at": "2026-07-17T12:28:13.601387",
  "duration": 2604.8065,
  "transcribed_at": "2026-07-17T13:01:02.335633" }
```

It remains **non-authoritative** — the database is what the application reads — but it
means a lost or deleted `earshot.db` costs nothing permanent. The `"encoded"` literal is
retained for `earshot-tui` compatibility and, since the encode is real, is now accurate.

## Reconciliation

Sessions are born in the database, so the two stores can disagree only through loss or
outside interference. On startup:

| Situation | Action |
|---|---|
| Row and directory both present | Normal. Nothing to do. |
| Row with **no directory** | The recording failed at creation, or the directory was removed outside the app. Mark the session missing; do not resurrect it. |
| Directory with **no row** | The database was rebuilt or lost. **Adopt it** — read `status.json` and insert a row. |
| Directory with chunks but no `session.m4a` | An interrupted session. Finalize it (below), then adopt or update. |

After adopting, set `sqlite_sequence` to the highest ID found on disk so a rebuilt
database can never re-issue an ID that already exists.

## Crash recovery

For each session directory whose finalization did not complete:

- If `session.m4a` exists, the session is already finalized.
- Otherwise, if `recording-NNN.wav` chunks are present, run the **same
  concatenate-and-encode pass** to produce `session.m4a`, then delete the chunks — the
  end-of-session path in
  [recording.md](recording.md#fr-3--fr-6-end-of-session--encode-to-one-m4a).
  - A chunk whose header was never finalized (crash before `close()`) is read tolerantly,
    taking the frame count from the file size.
  - If the encode fails — no disk space, for example — the chunks are **left in place** and
    logged; recovery is retried on the next boot. A single failure never aborts the scan of
    other sessions.

A job that was running at shutdown is handled by the queue, not here — see
[processing.md](processing.md#crash-resilience).

## Disk space management

- Disk usage is checked before each new recording and continuously while blocked.
- At or above `storage.disk_threshold_percent` (default 90%), the LED pulses **orange** and
  new recordings are blocked.
- The device recovers automatically once files are removed and usage drops below the
  threshold.
- A finished session is ~0.24 MB/min (AAC 32 kbps), so a 59 GB card holds roughly 230
  hours. Note the transient peak: during a session the uncompressed chunks are on disk at
  ~1.9 MB/min until the encode replaces them.
