# Processing

Transcription and diarization: what runs, where, and when. LED behavior:
[state-machine.md](state-machine.md). Both are initiated from the web UI
([web-ui.md](../requirements/web-ui.md), FR-24 / FR-25) — neither has a button gesture.

## Two routes, one of them optional

| | Runs where | Available |
|---|---|---|
| **Transcription** | on the Pi (faster-whisper), or on a processing service if one is configured | **always** |
| **Diarization** | only on a [processing service](../../service/README.md) | only when a service is configured and reports the capability |

**The device stands alone.** Out of the box, with no configuration, no network beyond the
LAN it serves its UI on, and no account anywhere, a Pi records and transcribes. That is
the product's baseline and nothing may weaken it
([ADR-0010](../adr/0010-optional-processing-service.md)).

**A service is an upgrade, never a dependency.** Setting `processing.service_url` routes
transcription to a machine better suited to it — far faster — and unlocks diarization,
which needs compute a Pi 4B does not have. Remove the URL and the device falls back to
local transcription; nothing breaks and nothing is lost.

> **Routing is a single rule.** If `processing.service_url` is set and the service is
> reachable, transcription jobs go there. Otherwise they run locally. Diarization has no
> local fallback: without a service the action is not offered.

## Processing jobs

- **At most one job at a time, device-wide.** The web UI disables the other action while
  one runs. Both write the same `transcript.md`, so this makes a concurrent-write conflict
  unreachable rather than merely unlikely, and keeps the **Processing** LED unambiguous.
- **Local transcription yields to recording; a service job does not.** This is the one
  place the two routes behave differently, and it follows from where the CPU is — see
  [Preemption](#preemption).
- **An interrupted job leaves no trace**, except a service job, which is resumable — see
  [Crash resilience](#crash-resilience).

## FR-14: Queue
- A session is **pending** when its directory has `session.m4a` but no `transcript.md`
  and no `.failed_processing` marker.
- The queue is implicit — derived from the filesystem at runtime (no queue file).
- Processed **FIFO**, lowest session ID first. Because IDs are allocated monotonically and
  need no clock, queue order is true capture order unconditionally
  ([storage.md](storage.md#session-identity)).
- The queue persists across reboots; a session stays pending until `transcript.md` or
  `.failed_processing` is written.
- Pending sessions are listed in the web UI for the user to run; a "transcribe all" action
  processes them one at a time, FIFO.

### FR-14a: Trigger
- Every job is initiated per session from the web UI. Nothing is submitted automatically.
- Diarize does not require a prior transcript — it produces one.

## FR-15: Process — local

The default path, used whenever no service is configured.

- The model is loaded once per run:
  `WhisperModel(model, device="cpu", download_root="~/.local/share/earshot/models", cpu_threads=threads)`.
  A load failure aborts the run.
- `session.m4a` is decoded and transcribed, reading lazily during segment iteration.
- Runs in a **cancellable worker thread** — a new recording cancels it (see
  [Preemption](#preemption)).
- On success: write `transcript_raw.json` then `transcript.md`, delete any
  `.processing_failures.json`, update `status.json` to `transcribed`.

Defaults are `transcription.model = "base.en"` and `transcription.threads = 2`, the latter
leaving headroom on the 4-core CPU. Expect roughly **7–13 minutes per 15 minutes of
audio** — so a 43-minute session takes 20–37 minutes. That is the cost of self-sufficiency,
and the reason a service is worth having if you have somewhere to run one.

## FR-15b: Process — service

Used whenever `processing.service_url` is set. The only route for diarization.

1. **Submit** `session.m4a` and the job kind to `POST {service_url}/v1/jobs`
   ([service API](../../service/specs/api.md#sr-1-post-v1jobs)).
2. **Persist the returned `job_id`** to `.job.json` *before* anything else.
3. **Poll** `GET /v1/jobs/{id}` every `processing.poll_interval_seconds` (default 5),
   surfacing the reported stage and progress. Progress is absent for stages the service
   cannot honestly measure; the UI shows the stage name instead.
4. On `done`, **fetch** the result and **render** the segments into `transcript.md`
   (FR-16), writing the raw response to `transcript_raw.json` — or
   `transcript_diarized_raw.json` for a diarize job.
5. **Delete** `.job.json` and any `.processing_failures.json`, update `status.json`.

The device renders the transcript either way. The service returns segments only and never
rendered text ([service ADR-0002](../../service/adr/0002-async-job-api.md)), because the Pi
is what knows the session's name, its speaker-name map, and the transcript format.

### Crash resilience
`.job.json` holds `{ "job_id": …, "kind": …, "submitted_at": … }`. On startup, for any
session directory containing one, the device **resumes polling that job** rather than
resubmitting — the work is already done or under way on the service.

If the service returns **404** for a persisted job id — reaped after its TTL, or lost to a
rebuild — delete `.job.json` and return the session to pending.

A *local* job has no such record: it holds no remote state, so an interrupted one simply
leaves the session pending and is re-run.

## Preemption

**Local transcription yields to recording.** It pegs CPU on the same machine that is
capturing, so a new recording (button or web) cancels it immediately; the session returns
to the front of the queue and recording begins without delay.

**A service job does not.** The work is on another machine and the Pi is holding an HTTP
connection, so a recording may start, run, and finish alongside it with neither degraded.

One rule, applied honestly: *whatever holds CPU on the Pi yields to capture.* Which route
is in use determines whether anything does.

## Failure
- A local failure, a submission failure, a `failed` job status, or an unreachable service
  is logged with the session id and attempt count. No transcript is written.
- The persisted count in `.processing_failures.json` (at least `count`, `last_failed_at`,
  `last_error`) survives restarts and reboots.
- If `processing.max_failures = 0`, the session stays pending and may be retried
  indefinitely. Otherwise, once the count reaches it, write `.failed_processing`, delete
  `.processing_failures.json`, and skip the session on future queue scans until the marker
  is removed — by the web UI's Retry action (FR-24) or manually.
- `session.m4a` always remains in place. A failure never costs audio.

> **An unreachable service is not a session failure.** If a configured service cannot be
> reached, the device reports a *connection* problem (FR-30) and does **not** increment
> any session's failure count. A LAN outage must not burn through retries on every pending
> recording — and transcription can fall back to running locally in the meantime.

## FR-16: Transcript format
`transcript.md`, `earshot-tui`-compatible:
```markdown
# <session name, or the session ID when unnamed>
**Session:** rec-000042
**Duration:** Xh Xm Xs
**Processed:** YYYY-MM-DD HH:MM:SS

---

[MM:SS] segment text
[HH:MM:SS] segment text
```
- **Header:** the user-assigned session name (FR-29) when one is set, otherwise the
  session ID. The transcript carries **no recording date** — a session is identified by
  its name or its ID, never by a wall-clock time it may not have had (see
  [Time independence](#time-independence)).
- **Session:** the session directory name, always present, as an opaque identifier that
  traces the transcript back to its directory.
- **Duration:** the `session.m4a` duration — derived from the audio, so it is correct
  regardless of the clock.
- **Processed:** wall-clock time the job completed.
- Timestamps use `[MM:SS]` under one hour, `[HH:MM:SS]` at/beyond one hour.
- Segment text is engine output, unmodified.
- **Renaming rewrites the header.** Assigning or changing a session name updates the `#`
  line of an existing `transcript.md` in place; nothing else in the file changes.

The format is identical whichever route produced it — a transcript does not record where
it was made.

### Time independence
Identity, ordering, and labelling never depend on the clock. Sessions are identified by an
allocated ID (`rec-NNNNNN`) that contains no date and is chosen without consulting the
clock, so a session captured on a Pi that has never had a valid time is indistinguishable
from any other — fully identifiable, nameable, orderable, and processable.

> `**Processed:**` is the only clock-derived field in the transcript. It is descriptive
> only — nothing reads it back — so a wrong clock degrades it without breaking anything.
> The capture time lives in `status.json` as `recorded_at`, which is likewise descriptive
> only ([storage.md](storage.md#time-is-metadata-not-identity)).

## FR-17: LED
- **Amber**, very slow pulse (~1.5–2 s) while a job runs; distinct from warning orange.
- The LED does **not** distinguish transcribe from diarize, or local from service. It
  reports the one locally actionable fact — the device is busy. During a recording the LED
  shows **Recording**; a service job running alongside is surfaced by the web UI.
- Returns to solid **green** when the job completes (no transition animation).

## Diarization (FR-25)

Diarization requires a processing service. There is no local path, and no cloud path: the
models need compute a Pi 4B does not have, and reaching for a third-party API to work
around that would trade the product's independence for a feature.

A diarize job returns the same segments as a transcribe job, each additionally labelled
`Speaker 1`, `Speaker 2`, … numbered by first appearance.

- **Labels are consistent across the entire recording**, however long it is. The service
  clusters over the whole audio in one pass
  ([service processing](../../service/specs/processing.md#diarize)), so a voice keeps one
  label from beginning to end. There is no chunking for the device to compensate for.
- **Names are assigned afterward, per session** (FR-27): the user plays a sample of each
  detected voice and names it, and the names are substituted into `transcript.md`. The map
  persists in `session.json`.
- Diarizing **overwrites** `transcript.md` and writes `transcript_diarized_raw.json`,
  whose presence marks the current transcript as diarized. There is only ever one
  `transcript.md` per session.
- Re-running a transcribe on a diarized session removes `transcript_diarized_raw.json` and
  the speaker labels, reverting it — including a *local* re-transcribe, which is how a
  diarized session is reverted when the service is gone.
- Offered only when the service reports `diarize: true` (FR-30). A service without
  diarization models still accelerates transcription.

## FR-18: Installer
- Installs `faster-whisper` via pip and pre-downloads the configured model to
  `~/.local/share/earshot/models/` (default `base.en` INT8, ~60 MB).
- `--no-transcription` skips the model download, for an installation that will always use
  a processing service. `transcription.enabled = false` has the same effect post-install.
- Writes `transcription.*` defaults and, if the user supplies one, `processing.service_url`.
