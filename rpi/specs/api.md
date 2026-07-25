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
One session, with its speaker map and current/last job when relevant. **404** if unknown.

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
The detected labels and any assigned names:

```json
{ "speakers": [ { "label": "Speaker 1", "name": "Ritchie", "segments": 14 },
                { "label": "Speaker 2", "name": null,       "segments": 9 } ] }
```

### `GET /v1/sessions/{id}/speakers/{label}/sample`
A short audio sample of that voice, drawn from the session, for the user to listen to
before naming. `Content-Type: audio/mp4`.

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
Bulk enqueue. `{ "kind": "transcribe", "target": "pending" }` enqueues every pending
session, oldest first ("transcribe all"). **202** with the created jobs.

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
Cancel. A `queued` job is dropped; a `running` local job is terminated
([job execution](../adr/job-execution.md)); a `running` service job is abandoned by the
device — it stops waiting on the connection. The stateless service cannot be cancelled, so
it finishes the in-flight `/asr` call and discards the result. **204**.

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
