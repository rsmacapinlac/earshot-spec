# Recording & Playback Spec

Audio capture and playback contract for earshot.

## Hardware constraint — mono only

Capture is **mono** — the board has a single ES8311 codec (one mic, one
speaker), so there is no stereo capture. The codec hardware details (config, I²S, the multi-mic caveat) are in the board reference: `../reference/audio-codec-es8311.md`.

## Audio format

| Property        | Value                                  |
| --------------- | -------------------------------------- |
| Sample rate     | 16 kHz                                 |
| Bit depth       | 16-bit signed PCM, little-endian       |
| Channels stored | 1 (mono)                               |
| Container       | WAV (RIFF/WAVE), 44-byte PCM header    |
| Data rate       | ~32 KB/s (~1.9 MB/min)                 |

## Recording

- **Codec capture:** opened as 16 kHz, **2-channel I²S**, with 32-bit slots
  carrying 16-bit audio in the MSB position; input gain 45.0.
- **Down-mix to mono:** keep the **left** slot only, converting each 32-bit slot
  to stored 16-bit PCM by shifting down from the MSB position. The right channel
  is a duplicate of the same mic and is discarded.
- **WAV header:** write 44 zero bytes first, stream PCM data, then `seek(0)` and
  back-fill the header once the length is known. Header fields:
  - `audioFormat = 1` (PCM), `channels = 1`, `sampleRate = 16000`
  - `byteRate = 32000` (sampleRate × blockAlign), `blockAlign = 2`
  - `bitsPerSample = 16`
- **Capture buffer:** 8 KB stereo I²S read buffer, allocated from heap. Because
  the link uses 32-bit stereo slots, each 8 KB read yields about 2 KB of stored
  16-bit mono PCM after keeping the left slot.
- **File naming:** one ID-named directory per recording,
  `/recordings/rec-NNNNNN/session.wav` (e.g.
  `/recordings/rec-000042/session.wav`); duration is stored beside it in
  `session.meta`. Optional voice labels are stored as `label.wav`. See
  `storage.md` for the full layout.
- **Validity floor:** a recording shorter than ~1000 bytes is discarded.

## Playback

- **Read:** skip the 44-byte header, read mono PCM in **1024-byte** chunks.
- **Mono → stereo:** duplicate each mono sample into both L and R for the codec
  (the DAC output path runs 2-channel).
- **Volume:** 85 during playback (codec out vol; 0 when idle).
- **Stop control:** polled mid-stream; a button press ends playback early.
- **Guard:** files of ≤ 44 bytes (header only, no audio) are rejected.

## Duration display format

Durations are rendered two ways, by context:

- **Live timers** — RECORDING, LABEL_CAPTURE, PLAYBACK (elapsed/countdown), and
  the DELETE_CONFIRM duration field — use full **`HH:MM:SS`** (e.g. `00:02:34`).
- **List durations** — the `RECORDINGS_LIST` per-note duration — use the compact
  **`M:SS`** form (e.g. `2:34`, `0:48`), no leading hour field.

Both derive from `duration_sec` / live elapsed seconds; the compact list form is
display-only and does not change stored metadata.

## Activity indicator (RECORDING / LABEL_CAPTURE)

The animated bars drawn during capture are a **decorative activity indicator**,
not a calibrated input-level meter. v1 does not require codec RMS/level metering;
the indicator may animate from input energy if cheap, but a simple looping
animation satisfies the contract. (Whether to invert the live treatment further
is **UX-4**.)

> Open technical decisions from this spec are centralized in
> `../requirements/open-technical-decisions.md`: the down-mix method (**TD-2**) and the
> 16 kHz / 16-bit quality ceiling (**TD-3**). Mono is a hardware limit, not a
> choice (see "Hardware constraint — mono only" above).

See also `../reference/device-rendering-constraints.md` and
`../requirements/non-functional.md`.
