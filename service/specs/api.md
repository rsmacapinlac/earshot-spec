# HTTP API

Normative contract for the processing service (SR-1 – SR-7). Asynchronous job model —
see [Asynchronous job API](../adr/async-job-api.md).

Base path: `/v1`. All responses are JSON except the audio upload, which is
`multipart/form-data`. No authentication in v1 (trusted LAN).

## Job lifecycle

```
POST /v1/jobs      -> 202 { job_id, status: "queued" }
GET  /v1/jobs/{id} -> 200 { status, progress?, error? }     (poll)
GET  /v1/jobs/{id}/result -> 200 { segments: [...] }        (once done)
DELETE /v1/jobs/{id}      -> 204                            (cancel or discard)
```

States are `queued` → `running` → `done` | `failed` | `cancelled`. `done`, `failed` and
`cancelled` are terminal.

## SR-1: `POST /v1/jobs`

`multipart/form-data`:

| Field | Type | Required | Notes |
|---|---|---|---|
| `audio` | file | yes | Any format ffmpeg decodes. earshot devices send `m4a`. |
| `kind` | string | yes | `transcribe` or `diarize`. |
| `language` | string | no | ISO code. Omit to auto-detect. |
| `num_speakers` | int | no | `diarize` only — a hint when the count is known. Omitted means the service infers it. |

Returns **202** with `{ "job_id": "...", "status": "queued" }`. The `job_id` is opaque;
callers must not parse it.

Errors: **400** unreadable or unsupported audio; **413** over `EARSHOT_MAX_UPLOAD_MB`;
**503** `kind` unavailable — check `/v1/health` (SNFR-4).

> **The response is immediate.** Acceptance means the audio was received and queued, not
> that it is valid speech or that processing will succeed.

## SR-4: `GET /v1/jobs/{id}`

```json
{ "job_id": "…", "kind": "diarize", "status": "running",
  "stage": "diarizing", "progress": 0.42,
  "submitted_at": "…", "started_at": "…", "finished_at": null,
  "error": null }
```

- `stage` — a short machine-readable label (`transcribing`, `diarizing`, `assigning`).
- `progress` — `0.0`–`1.0`, **present only when derivable from completed work**
  (SNFR-3). Absent during stages that cannot be measured; callers must render a stage
  name rather than assume a number exists.
- `error` — populated only when `status` is `failed`; a human-readable message plus a
  stable `error.code`.

**404** for an unknown or already-reaped job. A caller that gets 404 for a job it believes
is in flight must treat it as lost and resubmit.

## SR-5: `GET /v1/jobs/{id}/result`

**409** if the job is not `done`. Once `done`:

```json
{ "job_id": "…", "kind": "diarize", "duration": 2583.4,
  "language": "en",
  "segments": [
    { "start": 0.0,  "end": 5.8,  "text": "Okay, I think everyone's here.", "speaker": "Speaker 1" },
    { "start": 6.1,  "end": 13.9, "text": "Analytics look fine.",           "speaker": "Speaker 2" }
  ] }
```

- `start` / `end` are seconds from the beginning of the submitted audio.
- `speaker` is present for `diarize` and **absent** for `transcribe` — not null, absent.
- Speaker labels are `Speaker 1`, `Speaker 2`, … numbered by first appearance, and are
  **consistent across the entire recording** however long it is (SR-3, SNFR-2).
- The service returns **segments only**. It never returns Markdown or any rendered
  transcript — formatting belongs to the caller
  ([asynchronous job API](../adr/async-job-api.md)).

## SR-6: `DELETE /v1/jobs/{id}`

**204** always, whether the job was queued, running, terminal, or already gone —
idempotent by design so a caller cleaning up after a crash never has to reason about
state. A running job is cancelled; audio and results are discarded immediately.

## SR-7: `GET /v1/health`

```json
{ "status": "ok", "version": "…",
  "capabilities": { "transcribe": true, "diarize": false },
  "device": "cpu", "queue_depth": 2,
  "detail": { "diarize": "pyannote model not found on volume" } }
```

Callers gate their UI on `capabilities`, not on the service merely being reachable
(SNFR-4). `detail` explains any capability reported `false`.

## Retention

- **Audio is deleted as soon as a job reaches a terminal state** (SR-8), whether it
  succeeded, failed, or was cancelled.
- Results are retained for `EARSHOT_RESULT_TTL_HOURS` (default 24), then reaped along
  with the job record. Polling after that returns **404**.
- The service is not an archive; the device owns the durable copy.

## Restart behaviour (SR-9)

The job queue and job records survive restart on the state volume. On start-up, any job
recorded as `running` is marked `failed` with `error.code = "interrupted"` — it cannot be
resumed, because the audio for an in-flight job may be partially consumed, and reporting
failure honestly is better than a job that never terminates. `queued` jobs are retained
and run.

## Conventions

- Times are seconds (float) within the audio; timestamps are ISO-8601 UTC.
- Unknown request fields are rejected with **400** rather than ignored, so a caller
  learns immediately that an option did not take effect.
- Errors carry `{ "error": { "code": "...", "message": "..." } }`. Codes are stable;
  messages are not.
