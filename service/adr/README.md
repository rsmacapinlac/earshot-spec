# Architecture Decision Records (Processing Service)

Significant architectural and technical decisions for the earshot processing service,
with the context and reasoning behind them.

ADRs are referenced from other docs by title as well as number, so the numbering can
carry gaps (e.g. a retired ADR) without breaking references.

## Format
- **Status** — Proposed, Accepted, Deprecated, or Superseded
- **Context** — the problem that required a decision
- **Decision** — what was decided
- **Consequences** — trade-offs and implications
- **Alternatives** — what else was considered, and why it was not chosen

## Index

| ADR | Title | Status |
|---|---|---|
| [0001](0001-separate-processing-service.md) | Processing as a separate containerised service | Accepted |
| [0002](0002-async-job-api.md) | Asynchronous job API, returning segments | Accepted |
| [0003](0003-open-source-diarization.md) | Open-source diarization (pyannote), not a cloud API | Accepted |

Product requirements live in [`../requirements/`](../requirements/README.md); precise
implementation specs live in [`../specs/`](../specs/README.md).
