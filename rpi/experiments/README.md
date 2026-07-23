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
| [0001](0001-storage-bitrate.md) | Storage bitrate for `session.m4a` | [audio storage format](../adr/audio-storage-format.md) | Proposed |

Statuses: **Proposed** (designed, not run) · **Running** · **Complete** (outcome
recorded, decision made). Product requirements live in `../requirements/`; precise
implementation specs live in `../specs/`.

> An earlier experiment 0001 planned to validate OpenAI diarization quality on mono
> capture and cross-part speaker stitching (TD-7). It was replaced on 2026-07-21: TD-7
> was closed by decision — long sessions are split without attempting cross-request label
> continuity, and per-session speaker naming reconciles the labels — and diarization
> quality needs no release gate, being opt-in, off by default, and reversible. The number
> was reused because that experiment never ran and nothing referenced its results.
