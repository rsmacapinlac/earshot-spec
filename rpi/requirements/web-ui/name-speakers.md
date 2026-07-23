# Name the speakers

For each detected `Speaker N` in a diarized session, play a sample clip of that voice
drawn from the session and assign a name. Names replace the labels throughout that
session.

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

Applies only to diarized sessions — a plain transcript has no speaker labels to name.
See [diarize](diarize.md).
