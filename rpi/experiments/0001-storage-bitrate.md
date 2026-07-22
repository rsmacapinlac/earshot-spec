# 0001 — Storage bitrate for `session.m4a`

**Status:** Proposed (not yet run) · **Type:** hardware validation (non-normative)
**Related:** [`../adr/0001-audio-storage-format.md`](../adr/0001-audio-storage-format.md),
[`../specs/recording.md`](../specs/recording.md#fr-3--fr-6-end-of-session--encode-to-one-m4a),
[`../specs/configuration.md`](../specs/configuration.md)

**Decision this supports:** **ADR-0001** — whether **32 kbps AAC-LC** is the right stored
bitrate for `session.m4a`, or whether `recording.encode_bitrate_kbps` should ship higher.

**Decision rule:** Ship **32 kbps** if H1 and H2 pass. If either fails, ship **64 kbps**
and re-run. If 64 kbps also fails, the compression decision itself reopens — return to
storing PCM (the pre-2026-07-21 behaviour) and accept the ~8× storage cost.

## Why this needs measuring

The encode is **one-way and destructive**. Capture writes lossless PCM chunks, the encode
replaces them, and the chunks are deleted — there is no lossless copy to fall back on.
A bitrate that turns out too low degrades every recording made before the problem is
noticed, permanently. That asymmetry is what earns this an experiment: diarization
quality is a feature the user can switch off, but stored audio quality cannot be undone.

32 kbps was chosen on general grounds — it is comfortably inside normal practice for
16 kHz mono speech — not from measurement of *this* capture chain, which runs through the
WM8960 ALC speech preset with `ALC Max Gain` itself still provisional
([recording.md](../specs/recording.md#capture-front-end-wm8960-alc)). The two interact:
ALC lifts quiet passages and the noise floor with them, and a lifted noise floor is
exactly what a low-bitrate encoder spends bits on.

## Assumptions

If one of these proves false, the decision reopens.

- **A1 — capture format:** 16 kHz / 16-bit **mono**, one ReSpeaker mic, ALC speech preset
  applied and persisted ([recording.md](../specs/recording.md#capture-spec)).
- **A2 — encoder:** AAC-LC via `ffmpeg`, single concat-and-encode pass over the chunk
  list. Not HE-AAC, which behaves differently at these rates.
- **A3 — transcription engine:** faster-whisper `base.en`, the shipped default. Run every
  arm through the **same** engine and route — either all locally on the Pi or all on a
  [processing service](../../service/README.md) — since the comparison is between bitrates,
  not between routes.
- **A4 — the artifact is dual-purpose:** the same file is transcribed *and* listened to,
  so a bitrate must satisfy both. Machine accuracy alone is not sufficient evidence.

## Hypotheses

- **H1 (transcription accuracy):** WER on the 32 kbps encode is within **1.0 percentage
  point absolute** of the PCM control, on the same audio, transcribed by the same service
  and model.
- **H2 (listen-back):** on the 32 kbps encode, speech is fully intelligible with no
  artifacts a listener flags as distracting, across near and far talkers.
- **H3 (encode cost — informational):** a 43-minute session encodes in **≤ 60 s** on a
  Pi 4B. This validates ADR-0001's "tens of seconds" claim, which sizes the amber
  finalization window and is currently an estimate.

## Equipment

- Pi 4B + ReSpeaker 2-Mic HAT — the real capture chain, ALC preset applied. Do not
  substitute a USB mic or a file recorded elsewhere.
- A ~15-minute conversation with **at least two speakers at different distances** (one
  close, one across a table), plus a passage of quiet/low-level speech.
- A ground-truth transcript of that recording, typed by hand.

## Instrumentation

- **Retain the PCM.** The pipeline deletes chunks after encoding, so the control has to be
  preserved deliberately — copy `recording-*.wav` aside before finalization, or run the
  finalization step manually against a saved chunk set.
- One concatenated PCM master, then encodes at **24 / 32 / 64 kbps** from that same
  master, so every arm is bit-identical input.
- A WER script with fixed normalization (case, punctuation, filler words) applied
  identically to all arms.
- One transcription route used for all four arms — do not mix local and service results.
- `time` around the encode pass for H3.
- Hold constant: room, seating, ALC settings, transcription route, model version.

## Data to collect

| Quantity | Unit | How | Decides |
| --- | --- | --- | --- |
| WER — PCM control | % | faster-whisper vs. ground truth | H1 baseline |
| WER — 24 / 32 / 64 kbps | % | same, per arm | H1 |
| File size per arm | MB | `ls` | context for the trade |
| Listen-back rating | pass / flagged | blind playback, near and far passages | H2 |
| Encode wall-clock | s | `time` on the Pi 4B, 43-min input | H3 |

Derived: WER delta per arm against the PCM control; whether the far/quiet passages
degrade faster than the near talker (the case ALC most affects).

## Procedure

1. Record the ~15-minute session on the Pi with the shipped ALC preset. Preserve the
   `recording-*.wav` chunks.
2. Concatenate the chunks to a single PCM master. This is the control.
3. From that master, encode three arms: 24, 32, 64 kbps AAC-LC.
4. Transcribe all four (control + three arms) with the same engine and route. Compute WER
   against the hand-typed ground truth using identical normalization.
5. Blind listen-back on the 32 kbps arm versus the control — near-talker and
   far/quiet passages — noting any artifact a listener would call distracting.
6. Time the encode pass on a full-length (~43-minute) input for H3.

## Success criteria

- **H1:** 32 kbps WER within 1.0 pp absolute of the PCM control.
- **H2:** no distracting artifacts flagged at 32 kbps, near or far.
- **H3:** informational — report the measured encode time; flag if it exceeds 60 s, since
  the amber window is user-visible.

The 24 kbps arm carries no pass line. It exists to show where degradation begins — if 32
and 24 are indistinguishable, the chosen point is conservative; if 24 is clearly worse,
32 sits near the edge and 64 becomes the safer ship.

## Risks & confounders

- **ALC interaction.** `ALC Max Gain = 5` is itself provisional. If it is later raised, the
  lifted noise floor changes what the encoder must spend bits on and this result may not
  transfer — re-run if the ALC preset changes.
- **WER is not the whole story.** A transcript can score well while the audio is
  unpleasant to listen to. H2 exists precisely because the artifact is dual-purpose;
  do not let H1 alone decide.
- **Single recording.** One room and one pair of speakers. Prefer a second recording in a
  noisier setting before concluding, if the margin on H1 is thin.
- **Ground-truth labour.** Hand-typing a transcript for 15 minutes of speech is the main
  cost. Keep the recording short rather than skipping the control.

## Outcome

_To be filled in after running:_ measured WER per arm, listen-back result, encode
wall-clock, the bitrate shipped in `recording.encode_bitrate_kbps`, and whether ADR-0001's
finalization-window estimate held.
