# HTTP API

The device's one interface for **operating** it: the surface the web UI is a client of, and
the same surface any script or companion app uses
([the HTTP API is the interface](../adr/http-api-is-the-interface.md)). Every capability in
[`../requirements/web-ui/`](../requirements/web-ui/README.md) is an operation here.

> **First draft.** Endpoint shapes are settled enough to build against; field-level details
> will tighten as the implementation lands.

## Scope

- **Operating the device is here** — recordings, transcripts, recording control, jobs,
  device status, the processing-service connection.
- **Configuring the device is not.** Editing `config.toml` is an SSH operation
  ([configuration.md](configuration.md)); this API does not expose it. The single exception
  is the processing-service URL, an operational connection (below).
- **The on-disk layout is not a public contract.** Clients read recordings and transcripts
  through these endpoints, never by `scp` or by reading the session directory — so the file
  layout can change without breaking any client.

## Conventions

- Base path **`/v1`**. Bodies and responses are JSON unless noted; audio is binary.
- **No authentication in v1** — trusted LAN, matching the rest of the web surface.
- A session is addressed by its rendered id, `rec-NNNNNN`
  ([session identity](../adr/session-identity.md)).
- Times are ISO-8601; durations and positions are seconds (float); sizes are bytes.
- Errors carry `{ "error": { "code": "...", "message": "..." } }`. Codes are stable;
  messages are not.
- Unknown request fields are rejected with **400** rather than ignored.

## Device status

### `GET /v1/status`
The live device state — the same thing the LED reflects
([device status](../requirements/web-ui/device-status.md)).

```json
{ "state": "idle",
  "led": { "rgb": [0,255,0], "pattern": "solid" },
  "recording": { "session_id": "rec-000043", "elapsed": 128 },
  "processing": { "session_id": "rec-000042", "kind": "transcribe",
                  "route": "local", "stage": "transcribing", "progress": 0.42 },
  "disk": { "used_percent": 46, "blocked": false } }
```

- `state` — one of `booting`, `idle`, `recording`, `finalizing`, `processing`, `disk_full`.
- `recording` / `processing` are `null` when nothing is active. Both may be present at once
  (a service job runs while recording — [processing.md](processing.md#preemption)).
- `stage` / `progress` are present only for the **local** route, which the device runs and
  can measure. A **service** job is synchronous and opaque — the device omits both and the
  client renders an indeterminate *Processing* state
  ([off-the-shelf processing service](../adr/off-the-shelf-processing-service.md)).

### `GET /v1/events`
A **Server-Sent Events** stream (`text/event-stream`) so a client follows changes without
polling — including changes it did not cause, such as the hardware button being pressed.
This is what lets every open UI reflect the same state.

- On connect, one `state` event with the current `GET /v1/status` body.
- `state` — re-emitted whenever device state changes.
- `sessions-changed` / `jobs-changed` — a hint that the collection changed; the client
  refetches. The stream carries state and change hints, not full resource bodies.
- Clients reconnect on drop (native EventSource behaviour); no state is lost, since the
  first event after reconnect is a fresh snapshot.

## Sessions

### `POST /v1/sessions`
Create a session from an **uploaded** audio file
([upload an audio file](../requirements/web-ui/upload-audio.md)) — the second way a session
is born, alongside [recording](#post-v1recording).

`multipart/form-data`:

| Field | Type | Required | Notes |
|---|---|---|---|
| `audio` | file | yes | Any format ffmpeg decodes. Transcoded to the canonical `session.m4a` on ingest. |
| `name` | string | no | Optional label; omitted means unnamed (falls back to the ID). |
| `occurred_at` | string | no | Optional user-set date or date-time ([set a session date and time](../requirements/web-ui/set-session-datetime.md)). |

The device **accepts and encodes immediately**, returning **201** with the finished session once `session.m4a` exists. The session lands **pending**; processing is never auto-triggered.

Errors: **409** if a recording is active (upload is disabled while recording) or the disk threshold blocks new audio; **400** if the upload is unreadable or unsupported; **413** if it exceeds `storage.max_upload_mb`.

### `GET /v1/sessions`
All sessions, newest id first
([list sessions](../requirements/web-ui/list-sessions.md)).

```json
{ "sessions": [
  { "id": "rec-000042", "name": "Weekly sync — pricing",
    "occurred_at": "2026-07-20T14:00",
    "state": "diarized", "created_at": "2026-07-17T14:28:01",
    "duration": 2583.4, "size": 10289152,
    "has_transcript": true, "diarized": true } ] }
```

- `state` — `recording`, `pending`, `transcribed`, `diarized`, or `failed`, derived per
  [storage.md](storage.md#state--earshotdb). The UI's display wording ("Audio only",
  "Transcribed with Speakers") is a client concern.
- `name` is `null` when unset; clients fall back to `id`.
- `occurred_at` is the optional user-set session date/time
  ([set a session date and time](../requirements/web-ui/set-session-datetime.md)); `null`
  when unset. It is a user assertion, distinct from the clock-derived `created_at`, and is
  descriptive only.

### `GET /v1/sessions/{id}`
One session, its `speakers` map, and its current-or-most-recent **`job`** (or `null`).
**404** if unknown. The `job` is what the session view renders its processing status from
([view a session](../requirements/web-ui/session-detail.md)) — do not rely on `GET /v1/status`
for this, which reflects only the one *running* job device-wide and so is empty for a queued
session.

```json
{ "id": "rec-000042", "name": "Weekly sync — pricing", "occurred_at": "2026-07-20T14:00",
  "state": "pending", "created_at": "…", "duration": 1406.1, "size": 5760559,
  "has_transcript": false, "diarized": false, "speakers": [],
  "job": { "id": 23, "kind": "transcribe", "route": "service",
           "state": "queued", "stage": null, "progress": null,
           "attempts": 0, "last_error": null,
           "enqueued_at": "…", "started_at": null, "finished_at": null } }
```

- `job.state` is `queued`, `running`, `done`, `failed`, or `cancelled`. Queue **position**
  for a `queued` job is derived from the ordered [`GET /v1/jobs`](#get-v1jobs).
- `stage` / `progress` are populated only for a running **local** job; a **service** job
  reports neither (render an indeterminate Processing state).

### `PATCH /v1/sessions/{id}`
Set or clear the **name** ([name a session](../requirements/web-ui/name-session.md)) and/or
the **session date/time**
([set a session date and time](../requirements/web-ui/set-session-datetime.md)). Either
field may be present; omit a field to leave it unchanged.

```json
{ "name": "Weekly sync — pricing" }        // set/clear name  ({ "name": null } to clear)
{ "occurred_at": "2026-07-20T14:00" }      // set date+time
{ "occurred_at": "2026-07-20" }            // date only (time optional)
{ "occurred_at": null }                    // clear it
```

`occurred_at` is an ISO-8601 date or date-time; **400** if it is neither. Both fields apply
at any point in the session's life, including while recording, and rewrite the
`transcript.md` header if one exists. **200** with the updated session.

### `DELETE /v1/sessions/{id}`
Remove the session and everything in it, freeing disk
([delete a session](../requirements/web-ui/delete-session.md)). **204**. Confirmation is a
client concern. **409** if the session is currently recording. A processing job in flight
for it is cancelled, and its result discarded.

### `GET /v1/sessions/{id}/audio`
The `session.m4a`, streamed. Supports **Range** for in-browser seeking (**206** on a
ranged request). `Content-Type: audio/mp4`. On download, `Content-Disposition` uses the
session name when set, else the id
([play and download](../requirements/web-ui/play-and-download.md)). **404** before the
session is finalized.

### `GET /v1/sessions/{id}/transcript`
The transcript, content-negotiated:

- `Accept: text/markdown` → the rendered `transcript.md` (this is the export path).
- `Accept: application/json` → the segments: `[{ "start", "end", "text", "speaker"? }]`,
  `speaker` present only when diarized.

**404** if the session has no transcript yet.

## Speakers

Only meaningful for a diarized session
([name the speakers](../requirements/web-ui/name-speakers.md)).

### `GET /v1/sessions/{id}/speakers`
The detected labels, any assigned names, and each label's **voice samples** — the candidate
turns the UI offers for listening:

```json
{ "speakers": [
    { "label": "Speaker 1", "name": "Ritchie", "segments": 14,
      "samples": [ { "start": 61.2,  "end": 66.4, "text": "No, because I think it was ten bucks." },
                   { "start": 402.7, "end": 407.0, "text": "Right, and that's the piece I'd push back on." } ] },
    { "label": "Speaker 2", "name": null, "segments": 9, "samples": [ ] } ] }
```

- `samples` is ordered by `start` and holds **at most 5** turns. A sample's position in this
  array is the `n` taken by the sample endpoint below.
- `text` is that turn's transcript text, so a client can show what a sample says rather than
  offering audio alone
  ([name the speakers](../requirements/web-ui/name-speakers.md#hearing-a-voice)).
- `samples` is `[]` only when the label has no turns at all.

**Choosing the samples.** Candidates are that label's turns, filtered and ranked so the UI
offers a *representative* voice rather than an outlier:

1. **Drop turns shorter than 2 s** — too little audio to place a voice.
2. **Drop degenerate turns** — those whose distinct-word count over total-word count is
   below `0.5`. This is the shape of a transcriber stutter artifact
   (`"if we can, if we can, if we can, …"`), which is unusable as a sample and which a
   longest-turn rule selects for.
3. **Rank the rest by how close the turn's duration is to 4 s.** Longer is not better; a
   30 s turn truncated to its first 6 s is usually its least characteristic part.
4. **Spread the picks.** Take them in rank order, skipping any turn that starts within 60 s
   of one already taken, until 5 are held or candidates run out. Relax the 60 s spacing if
   fewer than 2 would otherwise be found.
5. **Never return nothing.** If every turn is filtered out, fall back to the label's single
   longest turn, so any label with turns offers at least one sample.

Selection is deterministic from the current transcript, so `n` is stable until the session
is re-transcribed.

### `GET /v1/sessions/{id}/speakers/{label}/sample`
The audio of one of that label's samples, for the user to listen to before naming.
`Content-Type: audio/mp4`.

- **`n`** (query, default `0`) selects the entry at that index in the label's `samples`.
  **404** for a label with no turns, or an `n` past the end of the array.
- The clip is exactly `[start, min(end, start + 6 s)]` of that turn. **It never runs past
  the turn's end**, so a sample can never contain a second voice — including for a turn
  shorter than the minimum clip length.
- **Cacheable** — `ETag` plus `Cache-Control: private`, the validator changing only when the
  transcript does. Each request is otherwise a fresh ffmpeg cut, and the UI replays samples
  freely while naming.

### `PUT /v1/sessions/{id}/speakers/{label}`
Assign or clear a name: `{ "name": "Ritchie" }` or `{ "name": null }`. Substituted into
`transcript.md`; persisted to the `speakers` table. **200**.

## Recording control

Shared with the hardware button — the API triggers the same state machine
([recording control](../requirements/web-ui/recording-control.md)).

### `POST /v1/recording`
Start a recording. **201** with the new session. **409** if a recording is already active,
or if the disk threshold blocks new recordings.

### `DELETE /v1/recording`
Stop the active recording; it finalizes to `session.m4a`. **200** with the finalized
session — or `{ "discarded": true, "reason": "too_short" }` when it was under the minimum
duration. **409** if nothing is recording.

## Jobs

The processing queue ([processing.md](processing.md#the-queue)). Transcription and
diarization are enqueued here; the queue is durable and drained one at a time.

### `POST /v1/sessions/{id}/jobs`
Enqueue a job for a session.

```json
{ "kind": "transcribe" }                      // or "diarize"
{ "kind": "diarize", "num_speakers": 3 }      // optional hint, diarize only
```

`num_speakers` is an optional hint for `diarize`, passed to the service as
`min_speakers = max_speakers = N` ([processing.md](processing.md#diarization)); omit it to
let the service infer the count.

**202** with the job. **409** if `kind` is `diarize` and no processing service reports the
capability ([diarize](../requirements/web-ui/diarize.md)), or if a job is already queued or
running for the session.

### `POST /v1/jobs`
Bulk enqueue, oldest first. **202** with the created jobs; sessions with a job already
queued or running are skipped.

- `{ "kind": "transcribe", "target": "pending" }` — every **pending** session (has audio,
  no transcript). This is *"transcribe all"* with the Diarize option **off**.
- `{ "kind": "diarize", "target": "undiarized" }` — every **not-yet-diarized** session (has
  audio, no diarized transcript), **independent of transcription state** — audio-only and
  already-transcribed alike. This is *"transcribe all"* with the Diarize option **on**;
  it overwrites plain transcripts with diarized ones. **409** if no processing service
  reports the diarize capability.

### `GET /v1/jobs`
The queue — queued, running, and recently finished:

```json
{ "jobs": [
  { "id": 128, "session_id": "rec-000042", "kind": "diarize", "route": "service",
    "state": "running", "stage": null, "progress": null,
    "attempts": 1, "enqueued_at": "…", "started_at": "…" } ] }
```

`stage` and `progress` are `null` for a running **service** job (synchronous, opaque); a
running **local** job may carry both.

### `GET /v1/jobs/{id}`
One job. **404** if unknown.

### `DELETE /v1/jobs/{id}`
Cancel a job the user no longer wants ([cancel a job](../requirements/web-ui/cancel-a-job.md)),
`queued` or `running`. A `queued` job is dropped; a `running` local job is terminated
([job execution](../adr/job-execution.md)); a `running` service job is abandoned by the
device — it stops waiting on the connection (the stateless service cannot be cancelled, so it
finishes the in-flight `/asr` call and discards the result). The job ends in the terminal
`cancelled` state and the session returns to **pending** — it is **not** requeued (unlike
[preemption](processing.md#preemption)). Idempotent: **204** whether the job was queued,
running, terminal, or already gone.

## Processing service

The one operational connection the API exposes — pointing the device at a
[processing service](../reference/processing-service.md)
([configure a processing service](../requirements/web-ui/processing-service.md)). This is
not general configuration: it changes where transcription runs and gates diarization, and
applies live.

### `GET /v1/service`
```json
{ "configured": true, "url": "http://homelab.local:9010",
  "reachable": true, "capabilities": { "transcribe": true, "diarize": true } }
```
`capabilities` are synthesized by the device from the service, not from the URL being set:
`transcribe` follows from reachability, and `diarize` from probing the service's
`/openapi.json` for the `diarize` parameter
([service API](../reference/processing-service.md#verifying)) — there is
no health endpoint.

### `PUT /v1/service`
Set the URL: `{ "url": "http://homelab.local:9010" }`. Applies immediately, no restart.
**200** with the refreshed status.

### `DELETE /v1/service`
Clear it — transcription falls back to local, diarization becomes unavailable. **204**.
