# NNNN — <Title>

**Status:** Proposed (not yet run) · **Type:** hardware validation (non-normative)
**Related:** `<links to the specs / requirements / ADRs this touches>`

**Decision this supports:** <the one open decision this experiment exists to settle —
name the TD or pending ADR, and the specific parameter(s) the data must pin down.>

**Decision rule:** <how the data maps to an outcome — e.g. "adopt X if it satisfies
all success criteria below; otherwise fall back to Y.">

## Assumptions

<Inputs and product requirements the experiment rests on. Number them (A1, A2, …)
and note that if one proves false, the decision reopens.>

- **A1 — <name>:** <assumption + why it matters / what it sets>.

## Background

<Why this is open, and what the proposed change is. Keep it brief.>

## Hypotheses

- **H1 (<name>):** <falsifiable statement with a target>.

## Equipment

<Hardware, instruments, stimulus. Note where a bench supply suffices vs. where a real
battery/load is required.>

## Firmware instrumentation (debug build)

<What the firmware must expose for the run: overridable parameters, log lines, GPIO
markers, things to suppress (e.g. e-paper refresh on unchanged state).>

## Data to collect

| Quantity | Unit | How | Decides |
| --- | --- | --- | --- |
| <quantity> | <unit> | <method> | <which parameter/hypothesis> |

Derived: <quantities computed from the measured set>.

## Procedure

### Test A — <name> (H?)
1. <step>

## Success criteria

- **H1:** <measurable pass line>.

## Risks & confounders

- <thing that could bias the result, and how to control for it>.

## Outcome

_To be filled in after running:_ <chosen parameter values, key measurements,
whether the change is adopted (and which fallback if not), and follow-ups.>
