# Transcribe a session

Initiate transcription on demand for a session, and show stage, progress, failures, and
retry.

**Always available.** Transcription runs on the device by default and needs no service,
account, or internet. Configuring a [processing service](processing-service.md) routes it
there instead — much faster — but removing the service falls back rather than breaking
([optional processing service](../../adr/optional-processing-service.md)).

- The UI shows **where** the work will run, since the two routes differ substantially in
  speed: roughly 20–35 minutes for a long session locally, a fraction of that on a
  service.
- **Progress is reported honestly.** Local transcription can report real progress from
  completed segments. A service reports its own stage, and omits a percentage for stages
  it cannot measure — in which case the UI shows the stage name rather than inventing a
  number.
- **Retry** enqueues a fresh job for a session whose previous one exhausted its attempts.
- A "transcribe all" action enqueues every pending session at once; the worker drains them
  one at a time, oldest first. The UI can show what is queued, not only what is running.

A local transcription is **cancelled** if a recording starts, because it holds CPU on the
capturing machine; a service job is not
([preemption](../../specs/processing.md#preemption)).
