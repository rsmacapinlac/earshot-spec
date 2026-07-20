# Open UX Questions

A living registry of interaction/UX decisions that are **deliberately unresolved**
— surfaced during design and requirements work but not yet settled. Nothing here
is canon; each entry notes an interim default so implementation isn't blocked.
When a question is decided, fold the outcome into the relevant requirement/spec
and drop it from this registry.

Each question has a stable ID (`UX-n`) so other docs can reference it.

| ID | Question | Status |
| -- | -------- | ------ |
| _None open._ | | |

Resolved questions are folded into the relevant spec/requirement and dropped from
this registry; see `../CHANGELOG.md` for the record. The v1 decisions were:

- **LED colour overload** — accepted for v1. The single LED reuses colours across
  states (amber = finalizing/transcribing, distinguished by pulse speed); the
  ambiguous cases are rare and low-stakes on a headless device. See
  [`../specs/state-machine.md`](../specs/state-machine.md).
- **Audio feedback** — out of scope; the LED is the sole feedback channel.
- **Single-button gestures** — kept as-is (short press = start/stop, 3 s idle hold
  = safe shutdown, no confirmation); shutdown only fires while idle and captures
  are always committed first, so an accidental shutdown is low-stakes. See
  [`../specs/state-machine.md`](../specs/state-machine.md) (FR-4).

---

> Note: **technical** (non-UX) open decisions live in their own registry,
> `open-technical-decisions.md`. This file is UX/interaction questions only.
