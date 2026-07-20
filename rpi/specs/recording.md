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
| Channels | **Stereo** — both ReSpeaker mics (`audio.channels = 2`) |
| ALSA PCM | `plughw:CARD=seeed2micvoicec,DEV=0` (`audio.alsa_pcm`) |
| Read block | 1024 frames per read |

Capture goes through ALSA against the configured `alsa_pcm`; `plughw:` performs
any needed rate/format conversion from the WM8960. Frames are written to a
`StereoWavWriter` (standard 16-bit PCM WAV, 44-byte header).

> **Stereo.** Sessions are recorded and stored **stereo** (both mics). Whether to
> downmix to mono — halving WAV size at little speech-to-text cost — is
> [TD-2](../requirements/open-technical-decisions.md#td-2--stored-wav-stereo-vs-mono).

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
16 kHz stereo PCM is ~3.8 MB/min, so a 43-minute session concatenates to a single
`session.wav` of ~165 MB.
