# Open UX Questions

A living registry of interaction/UX decisions that are **deliberately unresolved**
— surfaced during design and requirements work but not yet settled. Nothing here
is canon; each entry notes an interim default so implementation isn't blocked.
When a question is decided, fold the outcome into the relevant requirement/spec
and mark the entry **Resolved** (with the date and where it landed).

Each question has a stable ID (`UX-n`) so other docs can reference it.

| ID | Question | Status |
| -- | -------- | ------ |
| UX-1 | Overloaded LED colours (single-LED feedback) | Open |
| UX-2 | LED-only feedback with no audio cue | Open |
| UX-3 | Single-button gesture safety (hold-to-shutdown) | Open |

---

## UX-1 — Overloaded LED colours

**Status:** Open · **Related:** `../specs/state-machine.md`

The ReSpeaker HAT's single LED is the sole feedback channel, so distinct states
share a colour: **amber** covers both *encoding* and *transcribing* (differing
only by pulse rate), and **orange** covers both *disk full* and *USB transfer
error* (identical). A user cannot always tell what the device is doing from colour
alone. Should the states be made more distinguishable (distinct pulse patterns,
using more of the 3 available APA102 LEDs), or is the current overload acceptable
for a headless device?

**Interim default:** shared colours as specified; encoding vs. transcribing
distinguished only by pulse speed (~1 s vs. ~1.5–2 s).

## UX-2 — LED-only feedback, no audio cue

**Status:** Open · **Related:** `../requirements/backlog.md` (B-A1),
`../specs/state-machine.md`

Audio feedback is deferred to v2, so v1 confirms start/stop/complete/error purely
via LED. On a pocket device that may be out of sight, silent confirmation risks
the user not knowing a recording started or a transfer finished. Should a minimal
audible cue land sooner than v2?

**Interim default:** LED-only in v1; audio feedback deferred to v2.

## UX-3 — Single-button gesture safety

**Status:** Open · **Related:** `../specs/state-machine.md` (FR-4)

One button carries every action: short press = start/stop recording, 3 s hold
while idle = safe shutdown. There is no confirmation on shutdown, and the
short/long distinction is time-based. Is accidental shutdown (or an accidental
start/stop) a real risk that warrants a confirmation step or a different gesture?

**Interim default:** as-built — short press toggles recording, 3 s idle hold shuts
down, no confirmation.

---

> Note: **technical** (non-UX) open decisions live in their own registry,
> `open-technical-decisions.md` (e.g. capture gain, stereo vs. mono). This file is
> UX/interaction questions only.
