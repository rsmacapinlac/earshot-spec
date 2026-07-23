# Asynchronous Job API, Returning Segments

**Status:** Accepted (2026-07-21)

## Context

Two shapes were available for the API: hold the HTTP request open until processing
finishes, or accept the work and let the caller poll.

Transcribing a 43-minute recording takes minutes even on capable hardware — longer with
diarization, longer again on CPU. A synchronous request of that length is held open
across proxies and NAT timeouts, gives the caller nothing to display but a spinner,
and loses all record of the work if the connection drops.

A second question: should the service return a finished transcript, or structured data?

## Decision

**Jobs are asynchronous.** `POST /v1/jobs` returns a `job_id` immediately; the caller
polls `GET /v1/jobs/{id}` and fetches `GET /v1/jobs/{id}/result` when done.

**Results are segments, never rendered text.** The service returns `start`, `end`,
`text`, and — for diarization — `speaker`. Markdown, headers, timestamp formats, and
speaker names are the caller's concern.

Polling rather than callbacks: the caller is a device on a home LAN which may be behind
anything, and requiring it to be addressable would be a worse constraint than a poll
every few seconds.

## Consequences

- **A dropped connection costs nothing.** The `job_id` is the durable handle; a caller
  that crashes mid-job resumes by polling, provided it persisted the id. The Pi does
  exactly this.
- **Real progress can be reported** as work completes, rather than a spinner for several
  minutes.
- **Cancellation is expressible** — `DELETE /v1/jobs/{id}` — which a held-open request
  could not offer cleanly.
- **The caller must persist the `job_id`** to survive its own restart. A job whose id is
  lost is unreachable and eventually reaped; the caller resubmits. This is a real burden
  placed on every caller, accepted because the alternative loses the work outright.
- **Returning segments keeps presentation where the context is.** The device knows the
  session's name, whether the user renamed speakers, and what its transcript format is;
  the service knows none of that and should not guess. It also means a format change on
  the device needs no service release.
- **Two round trips to get a result** (status, then result) rather than one. Accepted:
  it keeps the polling response small and lets a caller poll frequently without
  transferring the payload each time.

## Alternatives

- **Synchronous request/response** — simplest possible client. Rejected: multi-minute
  requests are fragile across network equipment, offer no progress, no cancellation, and
  lose the work if the connection drops.
- **Webhooks / callbacks** — no polling. Rejected: requires the caller to be addressable
  from the service, which a device on a home LAN may not be, and adds retry and delivery
  semantics for a problem polling solves adequately at this scale.
- **Returning rendered Markdown** — the device would just write the file. Rejected: it
  puts presentation in the wrong place. The device would still need to re-parse it to
  substitute speaker names, and every transcript format change would become a service
  change.
- **WebSocket streaming of partial results** — nice progress UX. Rejected as premature:
  it complicates both sides for a job the user has already accepted will take minutes,
  and the polling contract can be extended later without breaking callers.
