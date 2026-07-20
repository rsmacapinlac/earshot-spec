# 0001 — OpenAI diarization on mono capture & cross-part speaker stitching

**Status:** Proposed (not yet run) · **Type:** hardware validation (non-normative)
**Related:** [`../requirements/web-ui.md`](../requirements/web-ui.md) (FR-25, FR-27),
[`../requirements/open-technical-decisions.md`](../requirements/open-technical-decisions.md) (TD-7)

**Decision this supports:** **TD-7** — whether the drafted long-session diarization
approach (compress → single request ≤25 min → split >25 min with
`known_speaker_references`) is viable on real ReSpeaker audio, and whether OpenAI
diarization is usable at all given **mono, closely-spaced-mic** capture.

**Decision rule:** Adopt the drafted approach and fold it into the diarization spec if
**H1–H3** pass. If H1 fails (mono quality too low), diarization is not worth shipping
on the current capture hardware — reopen with a hardware-capture change. If only H3
fails, ship diarization but **cap diarizable sessions at 25 min** (one request) and
drop cross-part splitting from v1.

## Assumptions

If one of these proves false, the decision reopens.

- **A1 — mono capture:** audio is 16 kHz/16-bit **mono**, one ReSpeaker mic
  (`recording.md` capture spec). Diarization has no stereo image to exploit.
- **A2 — local transcription unchanged:** faster-whisper remains the transcript engine;
  this experiment concerns the **diarize** path only.
- **A3 — verified OpenAI limits (2026-07-20):** `gpt-4o-transcribe-diarize`, 25 MB /
  1500 s per request, `chunking_strategy: "auto"` within a request, and cross-request
  label continuity only via `known_speaker_references` (undocumented workaround).
- **A4 — enrollment available:** named-speaker enrollment (FR-27) can supply 2–10 s
  reference clips per speaker.

## Background

TD-7: sessions can exceed OpenAI's per-request limits, and the model does not correlate
speaker labels across separate requests. The drafted fix compresses audio (so the
25-min duration cap binds, not the 25 MB size cap), sends ≤25-min sessions in one
request, and splits longer ones — reusing enrolled reference clips to keep a speaker's
label stable across parts. Two things are unproven: (1) diarization quality on mono,
closely-spaced-mic audio, and (2) whether reference clips actually hold identity across
a split boundary.

## Hypotheses

- **H1 (mono quality):** on a ~15-min, 2-speaker conversation, diarization attributes
  **≥ 90 % of correctly-transcribed speech time** to the right speaker (formal metric:
  DER ≤ 15 %).
- **H2 (single-request stability):** within one ≤25-min request, a given speaker keeps
  one consistent label for the whole session (0 mid-session flips).
- **H3 (cross-part stitching, enrolled):** for a >25-min recording split into ≤25-min
  parts, passing `known_speaker_references` keeps each speaker's label identical across
  the boundary in **≥ 4 of 5** split trials.
- **H4 (cross-part stitching, auto-carved):** reference clips auto-extracted from part 1
  (no manual enrollment) achieve cross-part stability within 1 trial of H3.

## Equipment

- Pi 4B + ReSpeaker 2-Mic HAT (the real capture chain — do not substitute a USB mic).
- A quiet-ish room; 2 speakers for H1–H3, a 3rd for a stretch case.
- An OpenAI API key with access to `gpt-4o-transcribe-diarize`.

## Instrumentation

- A script that: compresses `session.wav` to m4a, calls the diarize endpoint with
  `chunking_strategy: "auto"` and optional `known_speaker_names[]`/
  `known_speaker_references[]`, and saves the raw `diarized_json`.
- The ability to split a `session.wav` at a chosen timestamp into ≤25-min parts.
- **Ground truth:** a manual speaker timeline (who spoke when) for each test recording,
  for DER computation.
- Hold constant: model version, sample rate, mic gain (ALC preset), room, seating.

## Data to collect

| Quantity | Unit | How | Decides |
| --- | --- | --- | --- |
| Speaker attribution accuracy | % of speech time | diarized_json vs. ground truth | H1 |
| Diarization error rate (DER) | % | false-alarm + miss + confusion ÷ total speech | H1 |
| Mid-session label flips | count | inspect single-request output | H2 |
| Cross-boundary label match (enrolled) | pass/fail ×5 | compare label of each speaker either side of the split | H3 |
| Cross-boundary label match (auto-carved) | pass/fail | same, references carved from part 1 | H4 |
| Cost & wall-clock per session | USD, s | API response / timing | feasibility note |

Derived: per-speaker precision/recall; whether compression changes accuracy vs. raw WAV
(spot check on one recording).

## Procedure

### Test A — mono baseline quality (H1)
1. Record a ~15-min, 2-speaker conversation on the Pi 4B + ReSpeaker (natural turns,
   some short interjections). Annotate ground-truth speaker timeline.
2. Compress `session.wav` → m4a; diarize in one request (`chunking_strategy: "auto"`).
3. Compute attribution accuracy and DER against ground truth.

### Test B — cross-part stitching, enrolled (H2, H3)
1. Obtain a **>25-min** recording (record long, or concatenate real sessions with the
   same speakers). Enroll each speaker with a 2–10 s clip (FR-27).
2. Confirm single-request stability on a ≤25-min slice (H2).
3. Split into ≤25-min parts; diarize each part passing the enrolled
   `known_speaker_references`. Repeat with 5 different split points.
4. Record whether each speaker's label matches across every boundary.

### Test C — cross-part stitching, auto-carved (H4)
1. Same >25-min recording, **no** manual enrollment.
2. Diarize part 1; carve a clean 2–10 s clip per speaker from its segments.
3. Diarize part 2 passing those carved clips as references; check label continuity.

### Test D — stretch case (informational)
- Repeat Test A with 3 speakers and some overlapping speech; report accuracy/DER, no
  pass line.

## Success criteria

- **H1:** attribution accuracy ≥ 90 % (DER ≤ 15 %) on the 2-speaker recording.
- **H2:** 0 mid-session label flips in the single-request run.
- **H3:** labels match across the boundary in ≥ 4 of 5 split trials.
- **H4:** auto-carved references match enrolled performance within 1 trial.

## Risks & confounders

- **Mono / closely-spaced mics** are the core risk to H1 — little acoustic separation.
- **Overlapping speech** inflates DER; keep Test A/B turns mostly non-overlapping and
  isolate overlap in Test D.
- **Reference-clip quality:** clips with cross-talk or noise weaken H3/H4 — carve clean,
  single-speaker segments.
- **Ground-truth subjectivity:** annotate to a fixed convention; have one annotator do
  all recordings.
- **API nondeterminism / model updates:** record the model version; re-run a case to
  gauge variance.
- **Compression artifacts:** verify m4a vs. raw WAV doesn't materially change accuracy
  before trusting compressed uploads.

## Outcome

_To be filled in after running:_ measured accuracy/DER, label-stability results, whether
the drafted TD-7 approach is adopted (or the ≤25-min fallback), the compression format
settled on, and any follow-ups for the diarization spec.
