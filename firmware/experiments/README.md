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
| _None open._ | | | |

Statuses: **Proposed** (designed, not run) · **Running** · **Complete** (outcome
recorded, decision made). Product requirements live in `../requirements/`; precise
implementation specs live in `../specs/`.

> Experiment 0001 (timer-wake battery sampling during sleep) was removed in v1.7:
> sleep is buttons-only, so there is no autonomous battery check to validate. See
> `../specs/power-sleep.md` → "Sleep".
