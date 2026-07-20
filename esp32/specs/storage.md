# Storage Spec

This spec defines the v1 on-disk recording contract.

## Root

All recordings live under:

```text
/recordings/
```

The firmware creates this directory if needed.

## Recording directory layout

Each recording is one ID-named directory:

```text
/recordings/rec-NNNNNN/
  session.wav
  session.meta
  label.wav        # optional
```

Example:

```text
/recordings/rec-000042/session.wav
/recordings/rec-000042/session.meta
/recordings/rec-000042/label.wav
```

The directory name is the stable recording identity. v1 does not use date/time
for recording identity or metadata.

## ID allocation

- IDs use `rec-` followed by a zero-padded decimal sequence.
- Allocate IDs monotonically where possible, e.g. `rec-000001`, `rec-000002`, …
- A recoverable allocator file such as `/recordings/.next_id` may be used to
  avoid ID reuse across deletes.
- The allocator is not a manifest. If missing, unreadable, or lower than the scan
  result, recover by scanning existing `rec-*` directories and choosing `max + 1`.
- The SD-card scan remains authoritative for existing recordings.

### ID display form

The on-disk identity is the full 6-digit `rec-NNNNNN`. The UI shows the
human-readable short form **`REC NNN`** — uppercase, space-separated, with the
`rec-` prefix dropped and leading zeros trimmed to at least three digits (e.g.
`rec-000012 → "REC 012"`, `rec-001234 → "REC 1234"`). The short form is for
display only; the full directory name remains the stable identity.

## Files

### `session.wav`

The main recording audio. It is mono 16 kHz / 16-bit PCM WAV per
`recording-playback.md`. The WAV header is backfilled when capture stops.

### `session.meta`

Plain text app metadata. v1 requires at least:

```text
duration_sec=<seconds>
```

Do not write capture date/time metadata in v1.

If a voice label exists, metadata may also include:

```text
label_duration_sec=<seconds>
```

### `label.wav`

Optional short spoken label used to identify the recording. It uses the same WAV
encoding as `session.wav` unless a later spec changes label encoding. There is no
maximum label duration. The same ~1000-byte validity floor as `session.wav`
applies: a label capture below the floor is discarded and leaves any prior
`label.wav` untouched.

## Labels and relabeling

- Labels are optional.
- Skipping a label is not final; a label may be added later.
- Relabeling replaces the previous label. There is no label history.
- Replacement must be fail-safe: record the new label to a temporary file, then
  replace `label.wav` only after the new label is complete.
- If relabeling is cancelled, fails, or loses power mid-write, the previous label
  remains valid and the scan ignores the temporary file.

## Scanning and list building

On entering `RECORDINGS_LIST`, scan `/recordings`, validate recording
directories, sort by recording ID descending (newest allocated first), and cache
up to the newest 16 notes in the app model.

A directory is a valid recording only if it contains:

- a valid `rec-NNNNNN` directory name;
- `session.wav` with a valid WAV header and payload greater than 44 bytes; and
- `session.meta` with parseable `duration_sec`.

`label.wav` is optional. Invalid, partial, or corrupt directories are ignored so
interrupted captures are self-healing.

Each cached list entry records whether a valid `label.wav` is present, so the
list can render a **has-label** marker (speaker glyph) next to the ID.

### Counts and the 16-note cache

Two distinct counts are displayed:

- **MAIN `CLIPS`** is the full count of valid recordings on the card, which may
  exceed 16.
- The list's **`X OF Y`** position operates over the **browsable cached window**:
  `Y = min(total valid notes, 16)` and `X` is the 1-based selection within it.
  When more than 16 notes exist, the list browses the newest 16; `CLIPS` and the
  list's `Y` will then differ, by design.

## Commit and delete

A capture is committed only after:

1. `session.wav` has been fully written;
2. the WAV header has been backfilled; and
3. `session.meta` has been written.

Cancelled captures and captures under approximately 1000 bytes are discarded and
their directories removed.

Deleting a note removes its entire recording directory, including `session.wav`,
`session.meta`, and optional `label.wav`.

The `DELETE_CONFIRM` screen shows the note's ID, duration, and **size**. Duration
comes from `session.meta` (`duration_sec`); size is **derived** from the
`session.wav` file length at confirm time — it is not stored in metadata.
