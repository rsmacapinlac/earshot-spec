# Name and rename a session

Give a session an optional free-text name, and change it at any point in its life —
including while it is still recording.

**A name is a label, not an identity.** Identity is the allocated session ID `rec-NNNNNN`, which never changes ([session identity](../../adr/session-identity.md)). Names need not be unique — two sessions may both be "Standup" — and nothing looks a session up by name.

The name is used wherever the session is identified:

- As the primary label in the [session list](list-sessions.md), falling back to the ID.
- As the `#` header of `transcript.md`. Renaming rewrites that line in place; nothing else
  in the file changes.
- As the [download filename](play-and-download.md) for the audio.

Clearing the name reverts to the session ID everywhere. The name is stored on the session
record ([storage.md](../../specs/storage.md#state--earshotdb)) and mirrored to
`status.json`, which is what makes it survive a database rebuild.

> This is the same model the ESP32 track uses — `rec-NNNNNN` as stable identity with an
> optional human label on top — arrived at independently for the same reason: the device
> has no reliable clock, so identity cannot be a date.
