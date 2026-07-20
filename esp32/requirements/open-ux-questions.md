# Open UX Questions

A living registry of interaction/UX decisions that are **deliberately unresolved**
— surfaced during design and requirements work but not yet settled. Nothing here
is canon; each entry notes an interim default so implementation isn't blocked.
When a question is decided, fold the outcome into the relevant requirement/ADR
and mark the entry **Resolved** (with the date and where it landed).

Each question has a stable ID (`UX-n`) so other docs can reference it.

| ID | Question | Status |
| -- | -------- | ------ |
| UX-1 | Global PWR long-press "escape hatch" | Open |
| UX-4 | "You are live" treatment for RECORDING | Open |
| UX-7 | Advisory screens: auto-return vs keypress-dismiss | Open |

---

## UX-1 — Global PWR long-press "escape hatch"

**Status:** Open · **Related:** `../specs/state-machine.md`

Whether a long PWR press should be a universal "return to MAIN" gesture, and how
far it reaches. Sources disagree: the original spec proposed it everywhere except
RECORDING; the JSX prototype implemented it only in PLAYBACK/DELETE; the C++
prototype largely omitted it. Sub-questions:

- Is a hidden, unhinted global gesture desirable at all on a two-button device?
- Should it be discoverable (shown as a hint) or intentionally invisible?
- Does it collide with per-state long-press meanings (e.g. LIST's hold-to-delete)?

**Interim default:** don't wire PWR-long beyond the `LIST → DELETE_CONFIRM` case.

## UX-4 — "You are live" treatment for RECORDING

**Status:** Open

RECORDING currently signals "live" with a filled record dot + doubled header rule
+ big timer, but no inversion. Raised repeatedly in design and never decided:
should RECORDING get a stronger treatment (e.g. a fully inverted panel) so it's
unmistakable at a glance, or is the current treatment enough?

**Interim default:** keep the current record-dot + doubled-rule + big-timer treatment.

## UX-7 — Advisory screens: auto-return vs keypress-dismiss

**Status:** Open · **Related:** `../specs/state-machine.md`

The advisory condition screens (LOW BATTERY, CHARGING) are designed with a
`dismiss` keypress (PWR) that returns the user to the prior screen. Should they
instead **auto-return** after a few seconds with no keypress — so a low-battery
warning never strands the user mid-task — or is an explicit dismiss preferable so
the message can't be missed?

**Interim default:** keypress dismiss (PWR), as designed.

---

> Note: **technical** (non-UX) open decisions live in their own registry,
> `open-technical-decisions.md` (e.g. sleep depth, audio quality ceiling). This
> file is UX/interaction questions only.
