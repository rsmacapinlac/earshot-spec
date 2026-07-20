# 0001 — Capture gain: fixed PGA vs. WM8960 ALC

**Status:** Proposed (not yet run) · **Type:** hardware validation (non-normative)
**Related:** `../specs/recording.md` (Capture front-end — WM8960 ALC),
`../reference/respeaker-2mic-hat.md` (Capture front-end)

**Decision this supports:** finalizing the ALC capture front-end adopted in
[specs/recording.md](../specs/recording.md#capture-front-end-wm8960-alc) — chiefly
the **provisional `ALC Max Gain`** (5 vs 7) — and confirming the speech preset does
not regress transcription versus the shipped fixed-gain default. It also confirms
the ALSA index→dB mappings read back as expected on the device.

**Decision rule:** lock `ALC Max Gain` = 5 if it satisfies **all** success criteria
below; use 7 if it clears them with better distant-speech level and no worse
clipping/WER. Only fall back to the fixed +12 dB PGA if the ALC preset regresses
WER versus the shipped default.

## Assumptions

- **A1 — capture spec fixed:** 16 kHz / 16-bit / mono (left mic) capture from
  `plughw:CARD=seeed2micvoicec` is unchanged; only the WM8960 analog front-end
  (Capture PGA, ALC controls) varies. If capture format changes, results reopen.
- **A2 — transcription baseline:** `base.en` on Pi 4B (the default model) is the
  reference for WER comparison; a fixed reference script is read for every take.
- **A3 — same physical setup:** HAT seating, room, and mic orientation are held
  constant across takes so gain is the only variable.

## Background

The HAT ships with WM8960 `Capture` at 39/63 (+12 dB), `ADC PCM` at 0 dB, and
**ALC Function = Off**, **Noise Gate = Off**. Fixed gain means level depends
entirely on speaker distance/loudness: distant speech under-records, close speech
can clip. The spec adopts the Wolfson **speech** ALC preset
([recording.md](../specs/recording.md#capture-front-end-wm8960-alc)) to auto-level
toward a target; ALC can, if mistuned, add pumping/noise-floor lift. This
experiment validates the adopted preset on hardware and pins the provisional
`ALC Max Gain`.

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
| WER vs. reference transcript | % | `base.en` transcript vs. ground truth | H3 |

Derived: per-distance level spread (max−min RMS across distances) for fixed vs.
ALC — a lower spread means more consistent capture.

## Procedure

### Test A — shipped fixed-gain baseline (H1–H3)
1. Set Capture 39/63 (+12 dB), ALC Off, Noise Gate Off.
2. Record the reference passage at each of 0.3 / 1 / 2 / 3 m.
3. Log level, clip rate, noise floor, and WER per take.

### Test B — adopted ALC speech preset (H1–H3)
1. Apply the speech preset from `recording.md` (`ALC Function = Left`,
   `Target = 7`, `Attack = 2`, `Decay = 4`, `Hold = 0`, Noise Gate on, HPF on).
2. Repeat Test A's distance set at `ALC Max Gain = 5` and again at `= 7`.
3. Read back each control with `amixer … sget` to confirm the index→dB mapping.
4. Note any audible pumping or noise-floor lift.

## Success criteria

- **H1:** ALC median level at 2 m ≥ fixed +6 dB, noise floor ≤ −50 dBFS.
- **H2:** ALC clip rate at 0.3 m within +0.1 pp of fixed-gain baseline.
- **H3:** ALC mean WER ≤ fixed-gain mean WER across the distance set.

## Risks & confounders

- **Room/reader drift:** keep one session, one reader/source, marked distances.
- **ALC pumping:** audibly check and note; a WER win with obvious pumping is still
  a fail for conversation listening.

## Outcome

_To be filled in after running:_ the locked `ALC Max Gain`, the level/clip/WER
table, any mapping corrections, and the resulting update to
`../specs/recording.md` and `../reference/respeaker-2mic-hat.md`.
