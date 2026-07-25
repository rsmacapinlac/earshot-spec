# Processing

Transcription and diarization: what runs, where, and when. LED behavior:
[state-machine.md](state-machine.md). Both are initiated from the web UI
([transcribe](../requirements/web-ui/transcribe.md), [diarize](../requirements/web-ui/diarize.md)) — neither has a button gesture.

## Two routes, one of them optional

| | Runs where | Available |
|---|---|---|
| **Transcription** | on the Pi (faster-whisper), or on a processing service if one is configured | **always** |
| **Diarization** | only on a [processing service](../reference/processing-service.md) | only when a service is configured and reports the capability |

**The device stands alone.** Out of the box, with no configuration, no network beyond the
LAN it serves its UI on, and no account anywhere, a Pi records and transcribes. That is
the product's baseline and nothing may weaken it
([optional processing service](../adr/optional-processing-service.md)).

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
- **An interrupted job leaves no trace** and is simply re-run — the same on both routes.
  The service is stateless, so there is nothing to resume; the audio is on the device, so
  nothing is lost (see [Crash resilience](#crash-resilience)).

## The queue

Jobs live in the `jobs` table ([storage.md](storage.md#state--earshotdb)) — an explicit,
durable queue rather than a set inferred from which files happen to exist.

| Column | Meaning |
|---|---|
| `session_id` | the session being processed |
| `kind` | `transcribe` or `diarize` |
| `route` | `local` or `service`, decided at dequeue, not at enqueue |
| `state` | `queued` → `running` → `done` \| `failed` \| `cancelled` |
| `attempts`, `last_error` | retry accounting |
| `enqueued_at`, `started_at`, `finished_at` | timings |

- **One worker, one job at a time.** A single worker thread takes the oldest `queued` row,
  marks it `running` in the same transaction, and drains until the queue is empty. There is
  no broker and no task-queue framework — the table *is* the queue
  ([job execution](../adr/job-execution.md)).
- **Order is enqueue order**, by job id. Since session IDs are themselves monotonic and
  clock-free, a bulk enqueue of pending sessions is in true capture order.
- **The queue survives reboot.** A `queued` row is still queued after a restart.
- **Route is decided when the job starts**, not when it is enqueued — so a job queued while
  the service was unreachable still runs on the service if it has come back by then.

A session is **pending** when it has `session.m4a`, no `transcript.md`, and no unresolved
`failed` job. The web UI lists pending sessions; a "transcribe all" action enqueues them
all at once and the worker drains them.

### Trigger
- **Jobs are enqueued only from the web UI. Finalizing a recording never enqueues one.**
  A stopped recording becomes a pending session and waits; the user decides if and when to
  transcribe it. Processing is opt-in per session — the device does not spend 20–35 minutes
  of local CPU (or a diarization) on a recording nobody asked to transcribe.
- Diarize does not require a prior transcript — it produces one.
- A `queued` job can be **cancelled** outright. A `running` one follows the
  [preemption](#preemption) rules.

## FR-15: Process — local

The default path, used whenever no service is configured.

**Runs in a subprocess** ([job execution](../adr/job-execution.md)). The worker spawns a
child which loads the model and transcribes; the parent records the result.

- The child loads the model per job:
  `WhisperModel(model, device="cpu", download_root="~/.local/share/earshot/models", cpu_threads=threads)`.
  A load failure fails the job.
- `session.m4a` is decoded and transcribed, reading lazily during segment iteration.
- **Cancellation is termination.** A new recording kills the child (see
  [Preemption](#preemption)) — immediate and guaranteed, rather than depending on the
  transcriber noticing a flag between segments.
- **An OOM kill costs the job, not the recording.** The kernel takes the child; capture
  continues and the job is marked failed.
- On success: write `transcript_raw.json` then `transcript.md`, mark the job `done`, and
  update the session row and `status.json`.

Defaults are `transcription.model = "base.en"` and `transcription.threads = 2`, the latter
leaving headroom on the 4-core CPU. Expect roughly **7–13 minutes per 15 minutes of
audio** — so a 43-minute session takes 20–37 minutes. That is the cost of self-sufficiency,
and the reason a service is worth having if you have somewhere to run one.

## FR-15b: Process — service

Used whenever `processing.service_url` is set. The only route for diarization.

The service is **synchronous**: one request carries the audio and blocks until the
transcript returns ([off-the-shelf processing service](../adr/off-the-shelf-processing-service.md)).

1. **Submit** `session.m4a` to `POST {service_url}/asr` as multipart `audio_file`, with
   query params for the job — `output=json&encode=true&task=transcribe`, and for a diarize
   job `diarize=true` (plus `min_speakers`/`max_speakers` when a
   [speaker-count hint](#diarization) is set)
   ([service API](../reference/processing-service.md#adopted-service)). The worker holds
   the connection open for the full processing time.
2. On **2xx**, parse `segments[]`, keeping `{start, end, text, speaker?}` and **discarding**
   `words`, `word_segments`, and `language`.
3. For a diarize result, **normalize speaker labels** — map WhisperX's raw `SPEAKER_NN`
   (cluster id) to `Speaker N` by **first appearance** across the session (see
   [Diarization](#diarization)).
4. **Render** the segments into `transcript.md` (FR-16), writing the raw response to
   `transcript_raw.json` — or `transcript_diarized_raw.json` for a diarize job.
5. Mark the job `done`, update the session row and `status.json`.

A non-2xx response, a timeout, or a dropped connection fails the attempt
([Failure](#failure)); the service keeps no state, so there is nothing to reconcile.

Because the request is synchronous and opaque, a **service job has no stage or progress** —
the device surfaces it as an indeterminate *Processing* state, unlike local transcription,
which reports real progress. The device renders the transcript either way; the service
returns segments only and never rendered text
([off-the-shelf processing service](../adr/off-the-shelf-processing-service.md)), because the Pi is
what knows the session's name, its speaker-name map, and the transcript format.

### Crash resilience
On startup, any job left `running` is returned to `queued` and re-run — on **both** routes.
A synchronous service job holds no resumable remote state (the service is stateless), so an
interrupted one is simply submitted again. The audio always remains on the device, so a
reboot mid-job costs processing time, never data — and an interrupted re-run does **not**
count against `max_failures` ([Failure](#failure)).

## Preemption

**Local transcription yields to recording.** It pegs CPU on the same machine that is
capturing, so a new recording (button or web) terminates the inference subprocess; the job
returns to `queued` and recording begins without delay. Because cancellation is a signal
rather than a cooperative check, "without delay" is a contract rather than an aspiration.

**A service job does not.** The work is on another machine and the Pi is holding an HTTP
connection, so a recording may start, run, and finish alongside it with neither degraded.

One rule, applied honestly: *whatever holds CPU on the Pi yields to capture.* Which route
is in use determines whether anything does.

## Failure
- A local failure, a submission failure, a `failed` job status, or an unreachable service
  is logged with the session id and attempt count. No transcript is written.
- `attempts` and `last_error` are recorded on the job row and survive restarts.
- If `processing.max_failures = 0`, the job is re-queued indefinitely. Otherwise, once
  `attempts` reaches it, the job is marked `failed` and stops being retried; the session is
  no longer pending. The web UI's
  [Retry action](../requirements/web-ui/transcribe.md) enqueues a fresh job.
- `session.m4a` always remains in place. A failure never costs audio.

> **An unreachable service is not a session failure.** If a configured service cannot be
> reached, the device reports a *connection* problem ([service
> configuration](../requirements/web-ui/processing-service.md)) and does **not** increment
> any session's failure count. A LAN outage must not burn through retries on every pending
> recording — and transcription can fall back to running locally in the meantime.

## FR-16: Transcript format
`transcript.md`:
```markdown
# <session name, or the session ID when unnamed>
**Session:** rec-000042
**Duration:** Xh Xm Xs
**Processed:** YYYY-MM-DD HH:MM:SS

---

[MM:SS] segment text
[HH:MM:SS] segment text
```
- **Header:** the user-assigned session name ([session naming](../requirements/web-ui/name-session.md)) when
  one is set, otherwise the
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
Per [clock independence](../requirements/non-functional/clock-independence.md), nothing
here reads a timestamp back. Sessions are identified by an allocated ID (`rec-NNNNNN`)
containing no date and chosen without consulting the clock, so a session captured on a Pi
that has never had a valid time is indistinguishable from any other — fully identifiable,
nameable, orderable, and processable.

> `**Processed:**` is the only clock-derived field in the transcript. It is descriptive
> only — nothing reads it back — so a wrong clock degrades it without breaking anything.
> The capture time is the session's `created_at`, set when the recording started, and is
> likewise descriptive only ([storage.md](storage.md#state--earshotdb)).

## FR-17: LED
- **Amber**, very slow pulse (~1.5–2 s) while a job runs; distinct from warning orange.
- The LED does **not** distinguish transcribe from diarize, or local from service. It
  reports the one locally actionable fact — the device is busy. During a recording the LED
  shows **Recording**; a service job running alongside is surfaced by the web UI.
- Returns to solid **green** when the job completes (no transition animation).

## Diarization

Diarization requires a processing service. There is no local path, and no cloud path: the
models need compute a Pi 4B does not have, and reaching for a third-party API to work
around that would trade the product's independence for a feature.

A diarize job returns the same segments as a transcribe job, each additionally labelled
`Speaker 1`, `Speaker 2`, … numbered by first appearance.

- **Labels are consistent across the entire recording**, however long it is. The service
  clusters over the whole audio in one pass
  ([service processing](../experiments/0002-whisper-asr-webservice.md)), so a voice
  keeps one label from beginning to end. There is no chunking for the device to compensate
  for.
- **The device normalizes the labels.** WhisperX returns raw cluster labels — `SPEAKER_00`,
  `SPEAKER_01`, … numbered by **cluster id, not first appearance**. The device maps them to
  `Speaker 1`, `Speaker 2`, … by **first appearance** across the session (the first speaker
  heard becomes `Speaker 1`), and it is these normalized labels that appear in
  `transcript.md` and the `speakers` table. This mapping is the device's job because the
  synchronous service returns segments only
  ([off-the-shelf processing service](../adr/off-the-shelf-processing-service.md)).

  ```
  WhisperX segment (raw)                       ->  earshot Segment (normalized)
  {start:0.03, end:9.98,  speaker:"SPEAKER_01"} ->  {start:0.03, end:9.98,  speaker:"Speaker 1"}  # seen first
  {start:76.70,end:77.40, speaker:"SPEAKER_00"} ->  {start:76.70,end:77.40, speaker:"Speaker 2"}  # seen second
  # a transcribe-only run omits "speaker" entirely — never a null
  ```
- **A speaker-count hint is optional.** When the user supplies a count, the device passes it
  to the service as `min_speakers = max_speakers = N`
  ([speaker-count hint](../reference/processing-service.md#adopted-service)); with no hint, the
  service infers the count. It is a hint, not a guarantee.
- **Names are assigned afterward, per session** ([naming speakers](../requirements/web-ui/name-speakers.md)):
  the user plays a sample of each
  detected voice and names it, and the names are substituted into `transcript.md`. The map
  persists in the `speakers` table.
- Diarizing **overwrites** `transcript.md` and writes `transcript_diarized_raw.json`,
  whose presence marks the current transcript as diarized. There is only ever one
  `transcript.md` per session.
- Re-running a transcribe on a diarized session removes `transcript_diarized_raw.json` and
  the speaker labels, reverting it — including a *local* re-transcribe, which is how a
  diarized session is reverted when the service is gone.
- Offered only when the service reports `diarize: true`
  ([service configuration](../requirements/web-ui/processing-service.md)). A service without
  diarization models still accelerates transcription.

## FR-18: Installer
- Installs `faster-whisper` via pip and pre-downloads the configured model to
  `~/.local/share/earshot/models/` (default `base.en` INT8, ~60 MB).
- `--no-transcription` skips the model download, for an installation that will always use
  a processing service. `transcription.enabled = false` has the same effect post-install.
- Writes `transcription.*` defaults and, if the user supplies one, `processing.service_url`.
