# AGENTS.md

You are working in the **earshot spec repository**. Treat this repo as the source of
product requirements, firmware/software specs, ADRs, and hardware references for the
earshot project. See the [root README](README.md) for the repo overview — the product
tracks, their hardware, doc versions, and implementation repositories. This file covers
the working conventions for editing the docs.

## Documentation roles

Keep these boundaries between folders clear. Read the "Documentation
roles" in the core README.md.

Docs are organized by track — two device tracks (`esp32/`, `rpi/`); see the
[root README](README.md) for what each is. Device tracks own capture, storage, and
presentation. Heavier processing (faster transcription, diarization) is **optionally
offloaded to a processing service** on the LAN, and is never a dependency: a device must
remain fully useful without it. earshot does **not** build that service — it
[adopts an off-the-shelf one](rpi/adr/off-the-shelf-processing-service.md); the device owns
the integration, and the service is documented within the `rpi/` track (an ADR, an
experiment, and a deployment reference).

## Editing guidelines

- Keep links valid after moving/deleting files.
- After deleting or renaming docs, search the repo for stale references.
- Keep specs concise but testable.
- Keep ADRs focused on explaining the decision, alternatives that were explored, and consequences.
- **When an ADR is superseded or deprecated, move its file into a `superseded/` subfolder
  within that track's `adr/` directory** (e.g. `rpi/adr/superseded/`). Set its status
  line to `Superseded`/`Deprecated` with a link to the replacement, keep it non-normative
  and for history only, list it under a **Superseded** section in the track's `adr/README.md`,
  index it in `superseded/README.md`, and fix every inbound link to the new path.
- Keep reference docs factual and non-normative.
- Keep experiments hypothesis-driven and testable: name the design decision each
  one supports, state success criteria up front, and collect data against them;
  when an experiment closes the decision, update the spec/ADR/TD it informed.

Be concise, direct, and collaborative. The user is actively shaping product and firmware architecture; help clarify tradeoffs, then update the docs cleanly.

## Implementation repositories

Each track's code lives in its own GitHub repository (URLs and current doc versions
are in the [root README](README.md); per-track history is in each `CHANGELOG.md`).

Only inspect a local checkout of an implementation repository when explicitly asked to
compare these specs to the current implementation.

