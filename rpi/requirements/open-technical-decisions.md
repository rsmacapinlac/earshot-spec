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
| TD-1 | Capture gain: fixed PGA vs. WM8960 ALC | Open | Experiment 0001 |
| TD-2 | Stored WAV: stereo vs. mono | Open | — |
| TD-3 | Pi Zero 2W + USB-gadget offload scope | Open | — |
| TD-4 | Default transcription model on Pi 4B | Open | — |

---

## TD-1 — Capture gain: fixed PGA vs. ALC

**Status:** Open · **Related:** `../reference/respeaker-2mic-hat.md` (Capture
front-end tuning), `../specs/recording.md`, Experiment 0001

The WM8960 is configured with a **fixed +12 dB analog PGA and ALC (Automatic Level
Control) off**. Capture level is therefore static: distant/quiet speech is not
boosted, and close or loud speech can clip — both of which degrade transcription
accuracy. The codec's ALC is available and unused. Whether to enable ALC (and with
what target/max-gain/attack/decay), keep the fixed PGA, or retune the fixed gain
is unresolved.

**Path to closing:** measure recorded level and clipping across representative
speaker distances/levels with ALC off (current) vs. ALC on at a few settings, plus
the resulting transcription WER. See Experiment 0001.

**Interim default:** fixed +12 dB PGA, ALC off, noise gate off (as-built).

## TD-2 — Stored WAV: stereo vs. mono

**Status:** Open · **Related:** `../specs/recording.md`, `../specs/storage.md`

`session.wav` stores **stereo** from the two closely-spaced ReSpeaker mics. For
speech-to-text the second channel is largely redundant, so stereo doubles the WAV
size and offload footprint for little benefit. Whether to downmix to mono at
capture is unresolved.

**Interim default:** stereo (as-built capture).

## TD-3 — Pi Zero 2W + USB-gadget offload scope

**Status:** Open · **Related:** `../specs/usb-offload.md`,
`../requirements/hardware.md`

The design names a portable Pi Zero 2W target with USB-gadget-mode offload
(`g_mass_storage`), but the running implementation is **Pi 4B + USB-A stick only**,
and no gadget-mode code path was observed. Whether to build and specify the Zero 2W
path, or drop it and commit to Pi 4B, is unresolved.

**Interim default:** Pi 4B + USB-A offload only; the Zero 2W path is documented as
design intent, not specified normatively.

## TD-4 — Default transcription model on Pi 4B

**Status:** Open · **Related:** `../specs/transcription.md`

`tiny.en` is the default on both targets for portability, but the Pi 4B has ample
headroom for `base.en`, which is meaningfully more accurate (at ~2× the transcribe
time). Whether the Pi 4B default should switch to `base.en` — or the installer
should prompt per-device — is unresolved.

**Interim default:** `tiny.en` everywhere; `base.en` is opt-in via
`transcription.model` (backlog B-T2).
