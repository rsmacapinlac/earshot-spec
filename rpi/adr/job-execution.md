# Job Execution: In-Process Worker, No Task Queue Framework

**Status:** Accepted (2026-07-22)

## Context

Processing runs as jobs: transcribe or diarize, one session at a time. The work is **long** (20–35 minutes for a local transcription of a long session).

## Decision

**The queue is a table; the worker is a thread; local inference is a subprocess.**

- **No broker and no task-queue framework.** The `jobs` table in `earshot.db`
  ([state storage](state-storage.md)) is the queue. Standard library only: `sqlite3`,
  `threading`, `subprocess`.
- **One worker thread** in the main process takes the oldest `queued` row, marks it
  `running` in the same transaction, and drains until the queue is empty.
- **Local transcription runs in a subprocess.** The worker spawns a child that loads the
  model, transcribes, and returns segments; the parent records the result.
- **Service jobs stay in the worker thread.** They are HTTP requests and a poll loop, with
  nothing to isolate.

## Consequences

- **No daemon, no broker, no extra unit to supervise.** One systemd service, as before.
- **Enqueueing and updating a session are one transaction**, because the queue lives in
  the same database as the session. A separate broker would split that atomicity and
  introduce a queue that can disagree with the store.
- **Preemption becomes reliable.** Cancelling a local job is terminating a child process —
  immediate and guaranteed. In-process cancellation would have depended on the transcriber
  checking an event between segments, which makes "cancelled immediately, recording begins
  without delay" a hope rather than a contract.
- **An OOM kill costs the job, not the recording.** The kernel takes the child; the
  recorder keeps capturing and the job is marked failed.
- **Cost: the model is loaded per job, not per drain.** A subprocess cannot keep the model
  warm across jobs. Model load is seconds against 20–35 minutes of work, so this is
  negligible here — but it would not be if jobs were short and frequent.
- **Cost: results cross a process boundary** and must be serialized rather than returned as
  objects.
- **This does not scale sideways.** One worker, one machine, no parallelism, no
  distribution. That is the shape of the problem; if it changes, this decision should be
  revisited rather than stretched.
- When a processing service is configured, none of the isolation matters — nothing heavy
  runs on the Pi at all.

## Alternatives

- **Celery / RQ / Dramatiq / arq** — mature, well-understood, and all require a Redis or
  RabbitMQ daemon. Rejected: an unacceptable operational burden for a plug-in appliance,
  and enormous machinery for a handful of serial jobs a day.
- **Huey with `SqliteHuey`** — no broker, retries and result storage for free. Rejected
  more narrowly: it expects a separate consumer process, which splits LED and device-state
  ownership across two processes and adds coordination we do not otherwise need.
- **A framework of any kind.** These jobs are not generic tasks. The route is chosen at
  dequeue rather than enqueue; a local job is cancelled by a *physical button press*; a
  service job resumes after reboot by polling a remote identifier. None of that is
  expressible in a task queue's model, so it would be written by hand regardless — on top
  of a dependency that did not help.
- **Everything in one process, no subprocess** — simpler, keeps the model warm. Rejected:
  it leaves capture exposed to an OOM kill triggered by inference, and makes cancellation
  cooperative rather than guaranteed.
- **A separate long-running worker process** rather than a per-job child — keeps the model
  warm *and* isolates. Rejected for now: it needs IPC, its own supervision, and a story for
  which process owns the LED. Worth revisiting if per-job model loading ever becomes a real
  cost.
