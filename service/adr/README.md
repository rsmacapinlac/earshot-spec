# Architecture Decision Records (Processing Service)

Significant architectural and technical decisions for the earshot processing service, with
the context and reasoning behind them — and the alternatives that were rejected and why.

Each record gets its own file, named for the decision, and they are referred to **by name,
not by number**.

## Format
- **Status** — Accepted, Deprecated, or Superseded
- **Context** — the problem that required a decision
- **Decision** — what was decided
- **Consequences** — trade-offs and implications, including the costs
- **Alternatives** — what else was considered, and why it was not chosen

## Decisions

| Decision | In short |
|---|---|
| [Separate processing service](separate-processing-service.md) | Heavy processing runs in a container off the device — and is optional |
| [Asynchronous job API](async-job-api.md) | Submit, poll, fetch — returning segments, never rendered text |
| [Open-source diarization](open-source-diarization.md) | pyannote in the container, not a cloud API |

Product requirements live in [`../requirements/`](../requirements/README.md); precise
implementation specs live in [`../specs/`](../specs/README.md).
