# Set a session date and time

Give a session an optional date and time — when the conversation actually happened — and
change or clear it at any point, exactly like [naming a session](name-session.md).

**A date is a label, not an identity, and not the device clock.** Identity is the allocated
session ID `rec-NNNNNN`, which never changes ([session identity](../../adr/session-identity.md)).
This field, `occurred_at`, is a value the **user asserts** — not read from the Pi, which has
no reliable clock ([clock independence](../non-functional/clock-independence.md)). It is
distinct from the session's `created_at`, the clock-derived capture stamp that may be wrong
and that nothing acts on.

## Why this exists

The Pi 4B has no real-time clock, so the automatic capture time can't be trusted and is
never shown as fact. Letting the user optionally supply the real date and time is the
trustworthy counterpart: the one date on a session that *is* reliable is the one a human
put there on purpose.

## Behaviour

- **Optional and free to change.** Set it, edit it, or clear it whenever — including while
  the session is still recording — the same model as the name.
- **Date and time are each optional.** A user may set a date alone (`2026-07-20`) or a date
  with a time (`2026-07-20 14:00`). Whatever is provided is what is shown; nothing is
  invented to fill the gap.
- **Shown wherever the session is presented:**
  - as secondary information in the [session list](list-sessions.md), preferred over the
    clock-derived capture date when set;
  - as a **`Date:`** line in the `transcript.md` header, when set. Setting, changing, or
    clearing it rewrites that line in place; nothing else in the file changes — the same way
    renaming rewrites the `#` line.
- **Descriptive only.** Like every timestamp on the device, nothing sorts, looks up,
  recovers, or establishes identity by it. Ordering stays by ID. A user-set date being
  present or absent changes only what is *displayed*
  ([clock independence](../non-functional/clock-independence.md)).

Stored on the session record ([storage.md](../../specs/storage.md#state--earshotdb)) and
mirrored to `status.json`, so it survives a database rebuild — again, like the name.

> This does **not** make the device clock correct, and does not need it to be. It is a
> per-session label a human chooses, not a time source; NTP and an RTC remain out of scope
> ([clock independence](../non-functional/clock-independence.md#not-in-scope)).
