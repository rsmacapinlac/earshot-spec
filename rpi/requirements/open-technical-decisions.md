# Open Technical Decisions

A living registry of engineering decisions that are deliberately unresolved or
rest on assumptions worth revisiting — the technical counterpart to
`open-ux-questions.md`. Nothing here is settled; each entry notes an interim
default so implementation isn't blocked. When one is decided, fold the outcome
into the relevant ADR/spec and mark the entry **Resolved** (with the date and
where it landed).

Each has a stable ID (`TD-n`). "Owner" names an ADR or experiment when one already
frames the decision; standalone entries have none yet.

| ID | Decision | Status | Owner |
| -- | -------- | ------ | ----- |
| _None open._ | | | |

> TD-5 (**a single `transcript.md`; diarization overwrites it in place** — no separate
> diarized file; this supersedes the earlier two-artifact resolution) was re-decided on
> 2026-07-21. TD-6 was **re-decided on 2026-07-21**: speaker naming is **post-hoc and
> per-session** — diarization returns generic `Speaker N` and the user names each voice
> afterward from a sample clip. Enrollment is **out of scope**, superseding the
> 2026-07-20 resolution that put it in. Both are folded into [web-ui.md](web-ui.md).
>
> TD-7 (**long-session audio upload to OpenAI**) was **resolved on 2026-07-21**, and then
> **dissolved entirely** later the same day when processing moved to the
> [processing service](../../service/README.md): the service diarizes a whole recording in
> one pass, so there is no per-request limit, no split, and no cross-part label problem to
> decide about. The registry is empty.

## TD-7 — Dissolved 2026-07-21 (superseded by ADR-0010)

> **This decision no longer applies.** Its entire subject — OpenAI's per-request limits —
> disappeared when diarization moved to open-source models on the processing service
> ([rpi ADR-0010](../adr/0010-optional-processing-service.md),
> [service ADR-0003](../../service/adr/0003-open-source-diarization.md)). Kept as the
> record of a constraint that shaped the design for a day, and of how it was removed
> rather than worked around.

### Original resolution: split, don't stitch

**Verified limits** (`gpt-4o-transcribe-diarize`, checked 2026-07-20): **25 MB** per file
and **1500 s (~25 min)** per request; `chunking_strategy: "auto"` handles chunking
**within** a single request, keeping speaker identity consistent inside it. Labels are
**not** correlated **across** separate requests.

Sessions are stored as AAC 32 kbps m4a ([ADR-0001](../adr/0001-audio-storage-format.md))
at ~0.24 MB/min, so a full 25-minute request is ~6 MB. Size never binds; the **25-min
duration cap** is the only ceiling.

**Resolution:**
1. `session.m4a` is uploaded as stored — no compression step.
2. Sessions ≤ 25 min → one request with `chunking_strategy: "auto"`.
3. Sessions > 25 min → split into ≤25-min parts, each diarized independently. **No
   attempt is made to correlate speaker labels across parts.** A speaker may appear as
   `Speaker 1` in one part and `Speaker 3` in another; the UI lists every detected label
   and the user names them (FR-27), which is what reconciles identity.

**Why not carry labels across parts.** The only mechanism available is
`known_speaker_references` — an undocumented, community-reported workaround. It was
originally underwritten by speaker enrollment; when TD-6 made naming post-hoc, the clips
would have had to be auto-carved by the application from part 1, unvetted, with no
fallback if that failed. Post-hoc naming already solves the problem the references were
solving, at the cost of naming a few extra labels on a split session — a
deterministic, user-visible reconciliation instead of a speculative, invisible one.

**Consequences.** No dependency on undocumented API behaviour; no clip-selection rule to
specify or tune; long sessions remain diarizable. The cost is that a split session shows
more `Speaker N` entries than there are people, and the user names each. Diarization
quality on mono, closely-spaced-mic capture is **not** gated by a decision — the feature
is opt-in, off by default, requires a user-supplied key, and is reversible by
re-transcribing, so the user judges it directly.

Resolved decisions are folded into the relevant spec/ADR and dropped from this
registry; see `../CHANGELOG.md` for the record.
