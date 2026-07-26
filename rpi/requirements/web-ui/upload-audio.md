# Upload an audio file

Create a new session from an existing audio file, instead of recording it live — for
audio captured elsewhere (another recorder, a phone, an exported call).

**An upload is just another way to get a `session.m4a`.** Once ingested, an uploaded
session is indistinguishable from a recorded one — same directory, same `status.json`, same
identity, and the same transcribe / diarize / name / date / play / download / delete paths.
Nothing downstream needs to know how the audio arrived.

## Behaviour

- **Creates a new session** with the next allocated ID `rec-NNNNNN`
  ([session identity](../../adr/session-identity.md)), exactly as recording does.
- **The upload is transcoded to the canonical `session.m4a`** (AAC-LC, 16 kHz mono) on
  ingest — the same encode a recording ends with
  ([recording.md](../../specs/recording.md#fr-3--fr-6-end-of-session--encode-to-one-m4a)),
  so playback, download, size, and storage all behave identically. Any format ffmpeg can
  decode is accepted.
- **Accepted and encoded immediately.** The device ingests and encodes the file in the
  request, returning the finished session — surfaced as the existing **Finalizing (encode)**
  state while it runs ([state machine](../../specs/state-machine.md#led-states)).
- **Disabled while a recording is active.** Capture is sacred and the encode holds CPU on
  the Pi, so an upload is refused while recording; try again once idle.
- **Blocked when the disk threshold is reached**, like a new recording.
- **Lands pending (audio-only).** Processing is never auto-triggered
  ([processing.md](../../specs/processing.md#trigger)); the user then transcribes or
  diarizes it from the UI like any other session.
- **Not subject to the recording minimum-duration discard** — an upload is deliberate, so a
  short clip is kept.

## Optional metadata (neither required)

The upload may carry, and the form may prompt for:

- an optional **[name](name-session.md)** — otherwise the session is unnamed and falls back
  to its ID, as a recording does;
- an optional **[date and time](set-session-datetime.md)** (`occurred_at`). This is
  especially apt for an upload: the device's `created_at` is merely when the file was
  uploaded, which says nothing about when the conversation happened — so a user-asserted
  date is the only trustworthy one.

Both can also be set or changed afterward via the normal edit paths; supplying them at
upload is a convenience, not a requirement.

## Not in scope (v1)

- **Batch / multi-file upload.** One file, one session, per request.
- **Preserving the original file or its format.** The artifact is `session.m4a`; the upload
  is transcoded and not retained separately.

Contract: [`specs/api.md`](../../specs/api.md#post-v1sessions) (the endpoint),
[`specs/storage.md`](../../specs/storage.md#session-creation) (how the session is created).
