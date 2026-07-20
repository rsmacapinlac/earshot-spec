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
| TD-7 | Long-session audio upload to OpenAI diarization (per-request limits vs. multi-part sessions) | Open — approach adopted, validation pending | [web-ui.md](web-ui.md) + [exp 0001](../experiments/0001-openai-diarization-mono-and-chunking.md) |

> TD-5 (separate `transcript_diarized.md`, keep `transcript.md`) and TD-6
> (named-speaker enrollment **in scope**, FR-27) were resolved on 2026-07-20 and
> folded into [web-ui.md](web-ui.md).

## TD-7 — Long-session audio upload to OpenAI

**Verified limits** (`gpt-4o-transcribe-diarize`, checked 2026-07-20): **25 MB** per
file and **1500 s (~25 min)** per request; accepts wav/mp3/m4a/mp4/webm;
`chunking_strategy: "auto"` is required above 30 s and handles chunking **within a
single request** (speaker identity stays consistent inside one request). Speaker
labels are **not** correlated **across** separate requests — labels can flip.

**Earshot math:** mono 16 kHz/16-bit is ~1.9 MB/min, so a raw WAV hits the 25 MB cap
at ~13 min — *before* the 25-min duration cap. Compressing to a lossy format (m4a/mp3)
removes size as the binding limit, leaving the **25-min duration cap** as the real
ceiling per request.

**Adopted approach (validation pending — [exp 0001](../experiments/0001-openai-diarization-mono-and-chunking.md)):**
1. Compress `session.wav` to m4a/mp3 before upload so size stops binding.
2. Sessions ≤ 25 min → one request with `chunking_strategy: "auto"`; speaker identity
   is consistent, no stitching needed.
3. Sessions > 25 min → split into ≤25-min parts and hold speaker identity across parts
   by passing `known_speaker_references` — the enrolled clips from FR-27, or clips
   auto-carved from part 1 when speakers aren't enrolled. This cross-part workaround is
   community-recommended but **officially undocumented/experimental**.

**Still open:** validate the split-and-reference approach on real, mono ReSpeaker
audio (label stability across parts, and overall diarization quality on closely-spaced
mics) — [experiment 0001](../experiments/0001-openai-diarization-mono-and-chunking.md).
If validation fails on quality, diarization isn't viable on the current capture
hardware; if only cross-part stitching fails, cap diarizable sessions at 25 min.

Resolved decisions are folded into the relevant spec/ADR and dropped from this
registry; see `../CHANGELOG.md` for the record.
