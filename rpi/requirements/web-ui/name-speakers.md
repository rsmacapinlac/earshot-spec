# Name the speakers

For each detected `Speaker N` in a diarized session, listen to that voice and assign a
name. Names replace the labels throughout that session.

**Naming is per session and applied after the fact.** There is no enrollment step, no
cross-session speaker registry, and no identity carried between recordings — the same
person named in two sessions is named twice, independently.

Chosen because it needs no set-up before a meeting, and because naming a voice while
reading what it actually said is easier than naming it from a clip in the abstract.

- The mapping is stored per session ([storage.md](../../specs/storage.md#state--earshotdb))
  and mirrored to `status.json`, so it survives a reload, a restart, and a database
  rebuild, and can be corrected later.
- A label with no assigned name stays `Speaker N`.
- No names are ever sent anywhere; naming is entirely local relabelling.

## Hearing a voice

A voice sample is one **turn** taken from the session — a thing the speaker actually said,
not a clip cut at an arbitrary offset.

- **One sample is never enough.** Each speaker offers several turns to choose from, so a
  clip that lands on crosstalk or a filler phrase is never a dead end. A speaker with only
  one usable turn offers just that one.
- **Samples are representative, not the longest.** The longest turn is a bad sample: turns
  grow long because someone rambled or the transcriber merged noise into one block, and its
  opening seconds are the least characteristic part of it. Samples are drawn from that
  speaker's ordinary turns and spread across the session
  ([choosing the samples](../../specs/api.md#get-v1sessionsidspeakers)).
- **A sample contains only that speaker.** A clip never runs past the end of the turn it
  was cut from, so it cannot play a second voice and mislead the naming.
- **Only one thing plays at a time.** Starting a sample stops any other sample and any
  session playback already running.

Applies only to diarized sessions — a plain transcript has no speaker labels to name.
See [diarize](diarize.md).
