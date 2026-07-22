# earshot processing service — Requirements

What the service must do and the qualities that matter. Exact contracts live in
[`../specs/`](../specs/README.md); rationale in [`../adr/`](../adr/README.md).

Requirement IDs are `SR-n` (service requirement), distinct from the device tracks' `FR-n`.
IDs are never reused; retired requirements leave gaps.

## Capabilities

| ID | Capability |
|---|---|
| SR-1 | Accept an audio file plus a job type over HTTP and return a job identifier immediately, without holding the request open for the duration of processing. |
| SR-2 | **Transcribe** — produce time-stamped text segments for the submitted audio. |
| SR-3 | **Diarize** — produce time-stamped text segments each labelled with a speaker, consistent across the *whole* recording regardless of its length. |
| SR-4 | Report job state — queued, running, done, failed — and a progress indication where one can honestly be derived. |
| SR-5 | Return the completed result as structured segments. Rendering a human-readable transcript is the caller's responsibility, not the service's. |
| SR-6 | Accept a cancellation for a queued or running job. |
| SR-7 | Expose a health endpoint reporting whether the service is up and which capabilities are actually available (models present and loadable). |
| SR-8 | **Delete submitted audio once a job reaches a terminal state.** Audio is never retained beyond the job that needed it. |
| SR-9 | Survive restart without losing accepted work: a job accepted before a restart either completes or reports failure, and never silently disappears. |
| SR-10 | Deploy with `docker compose up`, configured by environment, with model weights on a persistent volume so restarts do not re-download them. |

## Non-functional

### SNFR-1 — No internet dependency
The service must run fully on a private network with no outbound access. Models are
fetched once at build or first run and cached on a volume; nothing at processing time
requires the internet. This is what keeps earshot local-first end to end.

### SNFR-2 — Long recordings are not a special case
A multi-hour recording is processed in one pass, with speaker labels consistent
throughout. There is no maximum duration and no chunking the caller must understand or
compensate for. Memory use must not scale linearly with duration.

### SNFR-3 — Honest progress
Progress is reported only where it can be derived from real work completed. A stage that
cannot be measured reports its stage name, not a fabricated percentage.

### SNFR-4 — Degraded, not broken
If diarization is unavailable — model missing, insufficient memory — transcription must
still work, and the health endpoint must say so. A caller can then offer the capability
that exists rather than failing both.

### SNFR-5 — Predictable resource use
Concurrency is bounded and configurable. The service must not accept unlimited
simultaneous jobs and exhaust the host; excess work queues.

## Out of scope (v1)

- **Authentication.** Trusted-LAN deployment, matching the device tracks' v1 posture.
- **Summarisation or any other LLM post-processing.** Transcription and diarization only.
- **Real-time / streaming transcription.** Jobs operate on complete files.
- **Speaker identity across recordings.** The service labels speakers *within* one
  recording as `Speaker N`. Naming them, and any notion of the same person recurring
  across sessions, belongs to the device.
- **Storage.** The service is not an archive. Audio is deleted after processing and
  results are retained only briefly for collection.
- **Multi-tenancy**, quotas, or per-caller isolation.
