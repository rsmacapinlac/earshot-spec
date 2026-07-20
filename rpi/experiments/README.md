# Experiments

Each experiment collects data to drive a **design decision**. An experiment names
the open decision it supports (a TD, or a pending ADR), states a hypothesis and
success criteria, runs a procedure that gathers measurements, and concludes with a
recommendation that updates the design. Experiments are non-normative and
time-bound — the evidence trail behind a decision. An experiment that does not lead
to a decision does not belong here.

Files are named `NNNN-slug.md`, numbered sequentially (like ADRs). Start a new one
from [`TEMPLATE.md`](TEMPLATE.md). When an experiment closes its decision, fold the
result into the relevant spec/ADR and update the related TD, leaving the experiment
as the record.

| # | Title | Decision it supports | Status |
| - | ----- | -------------------- | ------ |
| [0001](0001-openai-diarization-mono-and-chunking.md) | OpenAI diarization on mono capture & cross-part speaker stitching | TD-7 | Proposed |

Statuses: **Proposed** (designed, not run) · **Running** · **Complete** (outcome
recorded, decision made). Product requirements live in `../requirements/`; precise
implementation specs live in `../specs/`.
