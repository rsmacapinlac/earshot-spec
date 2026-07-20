# 0001 — Capture gain: fixed PGA vs. WM8960 ALC

**Status:** Proposed (not yet run) · **Type:** hardware validation (non-normative)
**Related:** `../requirements/open-technical-decisions.md` (TD-1),
`../reference/respeaker-2mic-hat.md` (Capture front-end tuning),
`../specs/recording.md`

**Decision this supports:** TD-1 — whether Earshot's ReSpeaker capture should keep
the as-built **fixed +12 dB PGA with ALC off**, or enable the WM8960's **ALC**
(and at what target / max-gain / attack / decay). The data must pin down the
capture chain's clipping rate and quiet-speech level across realistic speaker
distances, and the resulting transcription accuracy.

**Decision rule:** adopt an ALC configuration if it satisfies **all** success
criteria below (no worse clipping than fixed gain, higher usable level on distant
speech, and equal-or-better WER). Otherwise keep the fixed +12 dB PGA. If two ALC
settings both pass, prefer the one with lower WER, then lower clipping.

## Assumptions

- **A1 — capture spec fixed:** 16 kHz / 16-bit / stereo capture from
  `plughw:CARD=seeed2micvoicec` is unchanged; only the WM8960 analog front-end
  (Capture PGA, ALC controls) varies. If capture format changes, results reopen.
- **A2 — transcription baseline:** `tiny.en` on Pi 4B is the reference model for
  WER comparison; a fixed reference script is read for every take.
- **A3 — same physical setup:** HAT seating, room, and mic orientation are held
  constant across takes so gain is the only variable.

## Background

The device ships with WM8960 `Capture` at 39/63 (+12 dB), `ADC PCM` at 0 dB, and
**ALC Function = Off**, **Noise Gate = Off** (see reference). Fixed gain means level
depends entirely on speaker distance/loudness: distant speech under-records, close
speech can clip. ALC would auto-level toward a target, potentially improving both —
at the cost of pumping/noise-floor lift. This experiment measures whether ALC is a
net win for conversation recording + on-device STT.

## Hypotheses

- **H1 (quiet speech):** at ~2 m, ALC raises median speech level by ≥6 dB vs.
  fixed +12 dB without raising the idle noise floor above −50 dBFS.
- **H2 (no new clipping):** at ~0.3 m / raised voice, ALC clips no more than the
  fixed-gain baseline (clipped-sample rate within +0.1 pp).
- **H3 (accuracy):** ALC yields equal-or-better WER than fixed gain, averaged
  across the distance set.

## Equipment

- Pi 4B + ReSpeaker 2-Mic HAT (the `pi-earshot-pi4` unit).
- A fixed reference passage (~2 min) reproduced at a controlled level, or a live
  reader hitting marked distances (0.3 m, 1 m, 2 m, 3 m).
- `amixer -c <card>` to set WM8960 controls; `sox`/`ffmpeg` for level/clip stats.

## Instrumentation

- Set front-end per take with `amixer` (Capture PGA, `ALC Function`, `ALC Target`,
  `ALC Max Gain`, `ALC Attack/Decay`); record the exact control values in the take log.
- Save the WAV per take to measure true clipping and level.
- Hold `ADC PCM` at 0 dB and Noise Gate off across all takes.

## Data to collect

| Quantity | Unit | How | Decides |
| --- | --- | --- | --- |
| Peak / RMS level per take | dBFS | `sox <wav> -n stats` | H1 |
| Clipped-sample rate | % of samples at full-scale | `sox stats` / script | H2 |
| Idle noise floor | dBFS | RMS of a silent segment | H1 |
| WER vs. reference transcript | % | `tiny.en` transcript vs. ground truth | H3 |

Derived: per-distance level spread (max−min RMS across distances) for fixed vs.
ALC — a lower spread means more consistent capture.

## Procedure

### Test A — fixed-gain baseline (H1–H3)
1. Set Capture 39/63 (+12 dB), ALC Off, Noise Gate Off.
2. Record the reference passage at each of 0.3 / 1 / 2 / 3 m.
3. Log level, clip rate, noise floor, and WER per take.

### Test B — ALC sweep (H1–H3)
1. Set `ALC Function = Stereo`; for each of ~2–3 (Target, Max Gain) pairs, repeat
   Test A's distance set.
2. Note any audible pumping or noise-floor lift.

## Success criteria

- **H1:** ALC median level at 2 m ≥ fixed +6 dB, noise floor ≤ −50 dBFS.
- **H2:** ALC clip rate at 0.3 m within +0.1 pp of fixed-gain baseline.
- **H3:** ALC mean WER ≤ fixed-gain mean WER across the distance set.

## Risks & confounders

- **Room/reader drift:** keep one session, one reader/source, marked distances.
- **ALC pumping:** audibly check and note; a WER win with obvious pumping is still
  a fail for conversation listening.

## Outcome

_To be filled in after running:_ chosen front-end settings (fixed vs. ALC config),
the level/clip/WER table, whether ALC is adopted, and the resulting update to
`../reference/respeaker-2mic-hat.md`, `../specs/recording.md`, and TD-1.
