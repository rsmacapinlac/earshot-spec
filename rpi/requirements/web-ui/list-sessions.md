# List sessions

List all sessions with derived status, identity, duration, and size.

## Identity

A session is identified by its **name** ([session naming](name-session.md)) when one is
set, otherwise by its session ID. The ID is always available as the secondary identifier.

The capture date is shown as a scanning convenience only — never as identity. Nothing in
the interface requires the clock to have been right
([session identity](../../adr/session-identity.md)).

## Status

Read from the session record ([storage.md](../../specs/storage.md#state--earshotdb)):

| State | Displayed as |
|---|---|
| Recording in progress | Recording |
| Finalized, not yet processed | **Audio only** |
| Transcribed | Transcribed |
| Transcribed and diarized | **Transcribed with Speakers** |
| Processing failed | Failed |

The two bold labels are deliberate: "Audio only" says what you have rather than what you
lack, and is more useful than "pending" to someone deciding what to do next.
