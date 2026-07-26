# View a session

Open a single session and see everything about it in one place — and, crucially, **what is
happening to it right now** when a processing job is queued, running, or failed.

Most of the view composes capabilities specified elsewhere: the [name](name-session.md) and
[date](set-session-datetime.md) (both editable here), the audio [player and
download](play-and-download.md), the [transcript](transcribe.md) once it exists, the
[speakers](name-speakers.md) when diarized, and the [delete](delete-session.md) action. This
document specifies the piece that was missing: the **processing status** of the session.

## Processing status (the job overlay)

The view must reflect the session's current (or most recent) job, read from the session
itself (`GET /v1/sessions/{id}` returns a `job`), **not only** from device status — which
reflects the single *running* job device-wide and therefore shows nothing for a session that
is merely queued. A queued session must not look identical to an untouched one.

| Job state | The session view shows |
|---|---|
| **Queued** | A **Queued** indicator with its **position in line** (derived from the ordered queue, `GET /v1/jobs`), and a [Cancel](cancel-a-job.md) action. |
| **Running** | A **Processing** indicator. For a **local** job, real stage/progress; for a **service** job, an **indeterminate** Processing state (the synchronous service reports neither — [transcribe](transcribe.md)). Whether it will transcribe or diarize, and where it runs, is shown. A [Cancel](cancel-a-job.md) action is available. |
| **Failed** | The failure, with the stable error and a **Retry** ([transcribe](transcribe.md#behaviour)). |
| **Done / none** | No overlay — the resulting transcript is what's shown. |

- **It updates live.** The view follows the job through queued → running → done without a
  reload, via the [`/v1/events`](../../specs/api.md#get-v1events) stream (a `jobs-changed`
  hint prompts a refetch) — so a session watched while it sits in the queue visibly advances
  and then renders its transcript.
- **The session's own state is unchanged by all this.** A queued or running session is still
  `pending` (or `transcribed`, if being re-diarized) — the durable state is derived from
  artifacts; the job overlay is the transient live status on top of it
  ([list sessions](list-sessions.md), [storage.md](../../specs/storage.md#state--earshotdb)).

## Why this is called out

The data has always been on `GET /v1/sessions/{id}` (`job.state`), but no requirement said
the view had to render it — so a queued session could show nothing while a running one
showed activity, purely because the running one also appears in device status. This makes the
queued, running, and failed states first-class in the session view.
