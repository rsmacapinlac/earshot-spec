# AGENTS.md

You are working in the **earshot spec repository**. Treat this repo as the source of product requirements, firmware specs, ADRs, and hardware references for the earshot project.

## Documentation roles

Keep these boundaries clear:

- **Requirements** describe product/user needs and cross-cutting qualities.
- **Specs** define exact firmware behavior, file formats, thresholds, state transitions, and contracts.
- **ADRs** explain important architectural choices and alternatives. Do not duplicate full specs in ADRs.
- **Reference** documents hardware facts or non-normative bring-up/history notes.

When a doc contains exact implementation behavior, prefer placing it in `firmware/specs/`.
When a doc explains why a major approach was chosen, use `firmware/adr/`.

## Editing guidelines

- Keep links valid after moving/deleting files.
- After deleting or renaming docs, search the repo for stale references.
- Keep specs concise but testable.
- Keep ADRs focused on explaining the decision, alternatives that were explored, and consequences.
- Keep reference docs factual and non-normative.

Be concise, direct, and collaborative. The user is actively shaping product and firmware architecture; help clarify tradeoffs, then update the docs cleanly.

## Firmware Section

This firmware documentation is **v1.5**.

The firmware implementation lives in the separate GitHub repository:

```text
https://github.com/rsmacapinlac/earshot-firmware
```

Only inspect a local checkout of that repository when explicitly asked to compare these specs to the current firmware implementation.


