# Recording

Covers audio capture, chunk rollover, and end-of-session concatenation into a
single WAV (FR-2a, FR-3, FR-6). Storage layout and crash recovery are in
[storage.md](storage.md). Audio is stored as WAV — see
[ADR-0001](../adr/0001-audio-storage-format.md).

## Capture spec
| Parameter | Value |
|---|---|
| Sample rate | 16 kHz (`audio.sample_rate`) |
| Bit depth | 16-bit PCM (`audio.bit_depth`) |
| Channels | **Mono** — the left ReSpeaker mic (`audio.channels = 1`) |
| ALSA PCM | `plughw:CARD=seeed2micvoicec,DEV=0` (`audio.alsa_pcm`) |
| Read block | 1024 frames per read |

Capture goes through ALSA against the configured `alsa_pcm`; `plughw:` performs
any needed rate/format conversion from the WM8960. Frames are written to a mono
16-bit PCM WAV (standard 44-byte header).

> **Mono, single channel.** Sessions are stored **mono** from the **left** mic
> only. faster-whisper downmixes to mono for inference (so stereo adds no
> transcription value), the two mics are only ~cm apart (no usable stereo image
> without beamforming, which earshot does not do), and mono halves the WAV size.
> Capture the **left channel specifically** — do **not** average the two mics: a
> sum of two spatially-separated mics comb-filters whenever the talker is off-axis
> (the normal case), thinning the sound.

## Capture front-end (WM8960 ALC)

The WM8960's analog front-end is configured for **Automatic Level Control (ALC)**
using Wolfson's recommended **speech** preset, rather than a fixed input gain. ALC
tracks the input and adjusts the PGA to hold a target level — preventing clipping
on close/loud speech and lifting quiet/distant speech above the noise floor, both
of which improve listen-back and on-device transcription. The device is **not**
run with a fixed PGA and ALC off (the shipped default).

| Control | Value | WM8960 basis |
|---|---|---|
| `ALC Function` | `Left` | ALC on the captured (left) channel |
| `ALC Target` | `7` (≈ −12 dBFS) | speech-preset target level |
| `ALC Max Gain` | `5` (**provisional**) | capped below the datasheet's `7` (+30 dB) to limit noise-floor lift / pumping in near-field use |
| `ALC Min Gain` | `0` (≈ −17.25 dB) | |
| `ALC Attack` | `2` (≈ 24 ms, fast) | catches loud onsets → avoids clipping |
| `ALC Decay` | `4` (≈ 384 ms, slow) | avoids gain pumping |
| `ALC Hold Time` | `0` | |
| `Noise Gate` | `on` | stops ALC boosting pure silence |
| `ADC High Pass Filter` | `on` | removes DC / low-frequency rumble |

Applied via `amixer` against the `seeed2micvoicec` card:

```sh
card=seeed2micvoicec
amixer -c $card sset 'ALC Function'  Left
amixer -c $card sset 'ALC Target'    7
amixer -c $card sset 'ALC Max Gain'  5
amixer -c $card sset 'ALC Min Gain'  0
amixer -c $card sset 'ALC Attack'    2
amixer -c $card sset 'ALC Decay'     4
amixer -c $card sset 'ALC Hold Time' 0
amixer -c $card sset 'Noise Gate'    on
amixer -c $card sset 'ADC High Pass Filter' on
```

- **Persistence (required):** `seeed-voicecard.service` restores
  `/etc/voicecard/wm8960_asound.state` on boot, so these settings must be written
  there (`alsactl --file=/etc/voicecard/wm8960_asound.state store`) or they are
  lost on reboot. The installer applies and persists them (see
  [install-service.md](install-service.md#fr-8-one-line-install)).
- **`ALC Max Gain` is provisional.** The index→dB mappings above are derived from
  the WM8960 registers and should be read back on the device; the final Max Gain
  (5 vs 7) and confirmation of no transcription regression are validated by
  [Experiment 0001](../experiments/0001-capture-gain-alc.md).
- Front-end facts and the shipped default are in
  [../reference/respeaker-2mic-hat.md](../reference/respeaker-2mic-hat.md#capture-front-end).

## FR-2a: Chunked recording
- A session starts on button press. Its directory is named by start time:
  `recordings/<YYYYMMDDTHHMMSS>/`.
- Audio is written to sequentially numbered WAV chunks: `recording-001.wav`,
  `recording-002.wav`, …
- A chunk rolls over when its elapsed time reaches
  `recording.chunk_duration_seconds` (default 900 s): the current WAV is closed
  and the next chunk begins **without interrupting capture**.
- Rollover is timer-driven only. Chunks exist for in-session crash resilience
  (max loss = one chunk); they are concatenated at session end and then deleted
  (see FR-3).
- There is no maximum session duration. Recording continues until the button is
  pressed or the disk threshold is reached mid-session.
- Any chunk (including the final one) shorter than
  `recording.min_duration_seconds` is discarded rather than kept.

### Disk threshold mid-session
If the disk threshold is reached during recording, recording stops, the current
chunk WAV is closed, and the normal end-of-session concatenation is attempted. If
concatenation fails for lack of space, the chunk WAVs are retained for manual
recovery or retry on next boot.

## FR-3 / FR-6: End of session — concatenate to one WAV
On the button press that ends the session (LED → amber):

1. **Concatenate** all `recording-*.wav` chunks into a single **`session.wav`**
   (`concat_wav_files`; same-format PCM concat, no re-encode).
2. **Retain `session.wav`** — it is the artifact used for transcription and USB
   offload.
3. **Delete the `recording-*.wav` chunks** once `session.wav` is written, leaving
   one copy of the audio on disk ([ADR-0001](../adr/0001-audio-storage-format.md)).
4. Derive the session duration from `session.wav` (frame count ÷ sample rate) and
   write `status.json` (`status = "encoded"`, device hostname, `recorded_at`,
   `duration`).
   > The status literal `"encoded"` is retained for `earshot-tui` compatibility
   > even though no encode occurs; it means "capture finalized to `session.wav`."
5. Queue the session for transcription if enabled (see
   [transcription.md](transcription.md)); return to idle (green).

### FR-6a: Finalization failure
- A concatenation failure is logged to the journal.
- The `recording-*.wav` chunks are **retained** (not deleted) for manual recovery
  or next-boot retry.
- No LED feedback for finalization failure (post-recording is not a user-visible
  state beyond the amber window).

## Size reference
16 kHz mono PCM is ~1.9 MB/min, so a 43-minute session concatenates to a single
`session.wav` of ~83 MB.
