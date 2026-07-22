# Processing pipeline

How a job becomes segments (SR-2, SR-3). API contract: [api.md](api.md).

## Decode

All audio is decoded to 16 kHz mono PCM via ffmpeg before anything else, whatever was
uploaded. Devices send `m4a`; the service accepts anything ffmpeg reads so a caller is
never blocked on format.

## `transcribe`

faster-whisper (CTranslate2), model selected by `EARSHOT_WHISPER_MODEL`.

- Output is one segment per faster-whisper segment: `start`, `end`, `text`.
- Text is raw model output — **no post-processing**, no punctuation repair, no filler
  removal. A caller that wants cleanup does it downstream.
- Progress is `segment.end ÷ duration`, which is real completed work and therefore
  honest to report (SNFR-3).

## `diarize`

Transcription **and** speaker assignment. Three stages:

1. **Transcribe** — exactly as above. Progress reported.
2. **Diarize** — pyannote speaker-diarization produces speaker *turns*: time ranges with
   a speaker label, independent of the text. No meaningful progress is derivable, so this
   stage reports `stage: "diarizing"` with **no** `progress` field.
3. **Assign** — each transcript segment is given the speaker whose turns overlap it most
   by duration. A segment overlapping no turn keeps the nearest preceding speaker.

Labels are `Speaker 1`, `Speaker 2`, … numbered by first appearance in the recording.

> **Why the whole recording, in one pass.** pyannote clusters speaker embeddings across
> the entire audio, so a given voice gets one label from beginning to end no matter how
> long the recording is. This is the property a chunked cloud API cannot offer, and it is
> the reason diarization moved here — see
> [Open-source diarization](../adr/0003-open-source-diarization.md).

`num_speakers`, when supplied, is passed as a hint. It is not a constraint: a recording
that turns out to have more voices is not forced into the requested count.

## Models

| Purpose | Default | Env |
|---|---|---|
| Transcription | `base.en` | `EARSHOT_WHISPER_MODEL` |
| Diarization | `pyannote/speaker-diarization-3.1` | `EARSHOT_DIARIZE_MODEL` |

Weights live on the model volume ([deployment.md](deployment.md)). They are fetched once
— at image build or first run — and never at processing time (SNFR-1).

> **pyannote weights are gated.** The pretrained models require accepting terms on
> HuggingFace and a token to download. The operator does this once, at setup. If the
> weights are absent the service still starts, reports `diarize: false` on `/v1/health`
> with the reason, and serves `transcribe` normally (SNFR-4).

## Concurrency

- `EARSHOT_MAX_CONCURRENT_JOBS` (default 1) bounds simultaneous processing; further jobs
  queue (SNFR-5). Default 1 because both models are memory-hungry and a second concurrent
  job is more likely to exhaust the host than to finish sooner.
- Models are loaded once and held across jobs. First job after start pays the load cost.
- `EARSHOT_DEVICE` selects `cpu` or `cuda`. GPU is dramatically faster and entirely
  optional.

## Failure

A stage that raises marks the job `failed` with a stable `error.code`, leaving no partial
result — a job is `done` only if every stage completed. Audio is deleted on failure just
as on success (SR-8).

| `error.code` | Meaning |
|---|---|
| `decode_failed` | ffmpeg could not read the upload |
| `no_speech` | Decoded successfully, produced no segments |
| `model_load_failed` | Weights missing or unloadable |
| `oom` | Ran out of memory — reduce concurrency or model size |
| `interrupted` | Service restarted while the job was running (SR-9) |
| `cancelled` | Caller issued `DELETE` |
