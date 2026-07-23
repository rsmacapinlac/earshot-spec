# Recording

Covers audio capture, chunk rollover, and end-of-session encoding into a single
**`session.m4a`** (FR-2a, FR-3, FR-6). Storage layout and crash recovery are in
[storage.md](storage.md). Capture is lossless PCM; the retained artifact is
compressed — see
[Audio storage format](../adr/audio-storage-format.md).

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
- **`ALC Max Gain = 5` is the v1 implementation value.** Although this value is
  provisional pending hardware validation, implementers must ship `5` unless a
  documented experiment updates this spec. The index→dB mappings above are
  derived from the WM8960 registers and should be read back on the device during
  bring-up. Raise toward `7` only after measured evidence shows distant speech is
  under-levelled and the increase adds no audible pumping/noise-floor lift or
  transcription regression.
- Front-end facts and the shipped default are in
  [../reference/respeaker-2mic-hat.md](../reference/respeaker-2mic-hat.md#capture-front-end).

## FR-2a: Chunked recording
- A session starts on a button press or a web start. Its directory is named by
  the next allocated session ID: `recordings/rec-NNNNNN/` — no clock is consulted (see
  [storage.md](storage.md#session-identity)).
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
- `recording.min_duration_seconds` is a **session**-level minimum: a session whose
  total captured audio is shorter than it is discarded (LED double-flashes green,
  see [state-machine.md](state-machine.md#fr-2-start-recording)). A normal timer
  rollover is never subject to this check — a short final chunk of a longer session
  is encoded into the session like any other.

### Disk threshold mid-session
If the disk threshold is reached during recording, recording stops, the current chunk
WAV is closed, and the normal end-of-session encode is attempted. The encode *frees*
space overall — the m4a is ~1/8 the size of the chunks it replaces — but needs headroom
while both exist. If it fails for lack of space, the chunk WAVs are retained for manual
recovery or retry on next boot.

## FR-3 / FR-6: End of session — encode to one m4a
On the button press that ends the session (LED → amber):

1. **Concatenate and encode in a single `ffmpeg` pass** over the ordered chunk list
   (concat demuxer → AAC), producing **`session.m4a`** — AAC-LC at
   `recording.encode_bitrate_kbps` (default 32), 16 kHz mono. No intermediate
   full-length WAV is written, so peak disk during finalization is the chunks plus the
   growing m4a (~1.13× session size) rather than ~2×.
2. **Retain `session.m4a`** — it is the artifact used for transcription, playback,
   download, and diarization upload.
3. **Delete the `recording-*.wav` chunks** once `session.m4a` is complete, leaving one
   copy of the audio on disk (see
   [Audio storage format](../adr/audio-storage-format.md)).
4. Derive the session duration from `session.m4a`, update the session record, and write
   `status.json` (`status = "encoded"`, device hostname, name, speakers, `created_at`,
   `duration`) — see [storage.md](storage.md#the-database-is-rebuildable).
   > The status literal `"encoded"` was chosen for `earshot-tui` compatibility back when
   > no encode occurred. It is now literally accurate.
5. Queue the session for transcription if enabled (see
   [processing.md](processing.md)); return to idle (green).

### FR-6a: Finalization failure
- A concatenate/encode failure is logged to the journal.
- The `recording-*.wav` chunks are **retained** (not deleted) for manual recovery
  or next-boot retry, and any partial `session.m4a` is removed so the session reads
  cleanly as "not yet finalized".
- No LED feedback for finalization failure (post-recording is not a user-visible
  state beyond the amber window).

## Size reference
| Stage | Rate | 43-min session |
|---|---|---|
| `recording-*.wav` chunks (transient) | 16 kHz mono PCM, ~1.9 MB/min | ~83 MB |
| **`session.m4a`** (retained) | AAC-LC 32 kbps, ~0.24 MB/min | **~10 MB** |

The chunks exist only for the duration of the session, so ~10 MB is what a finished
session costs on disk — roughly 230 hours on a 59 GB card before the 90% threshold.

> **32 kbps is the v1 value, to confirm during bring-up.** It is chosen on general
> grounds for 16 kHz mono speech, not measured on this capture chain — and the encode is
> one-way, so a value that is too low degrades every recording made before it is noticed.
> Listen back to the first real sessions and compare against 64 kbps before accumulating
> a library; [experiment 0001](../experiments/0001-storage-bitrate.md) sets out the
> measurement. Raise via `recording.encode_bitrate_kbps` if speech is not clean.
