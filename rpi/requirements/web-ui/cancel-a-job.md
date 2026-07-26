# Cancel a job

Cancel a transcription or diarization job the user no longer wants — whether it is still
**queued** or already **running**.

The queue is visible ([transcribe](transcribe.md)), and any job in it can be cancelled from
where it is shown — the queue view, or the session it belongs to.

## Behaviour

- **A queued job is dropped.** It is removed from the queue and never runs. The session
  simply stays **pending** — its audio is untouched.
- **A running job is stopped.** A local job's inference is terminated immediately; a service
  job's request is abandoned by the device (the stateless service finishes its in-flight
  call and the result is discarded — [processing.md](../../specs/processing.md#fr-15b-process--service)).
  Either way the job ends in the terminal `cancelled` state and the session returns to
  **pending**.
- **Cancelling never costs audio.** `session.m4a` always remains; a cancelled session can be
  transcribed or diarized again later.
- **Idempotent and safe.** Cancelling a job that has already finished, failed, or is already
  gone is not an error — nothing happens.

## User-cancel is not preemption

These look similar but differ in outcome, and the distinction matters:

| | Trigger | Outcome |
|---|---|---|
| **Preemption** | A recording starts while a *local* job runs | The job is stopped and **returned to the front of the queue** to run later — the user still wants the transcript ([preemption](../../specs/processing.md#preemption)). |
| **User cancel** | The user cancels the job | The job ends **`cancelled`** and is **not** requeued — the user no longer wants it done. |

## Not in scope (v1)

- **Cancelling a recording** is a different action — that is [stopping it](recording-control.md),
  which finalizes the audio rather than discarding it.

Contract: [`specs/api.md`](../../specs/api.md#delete-v1jobsid) (`DELETE /v1/jobs/{id}`),
[`specs/processing.md`](../../specs/processing.md#the-queue) (the queue and job states).
