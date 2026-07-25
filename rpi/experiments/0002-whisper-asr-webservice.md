# 0002 — WhisperX processing service (ahmetoner/whisper-asr-webservice)

**Status:** Running — **blocked** (2026-07-25: the live deployment returns HTTP 500 on
every ASR request; H1–H3 cannot be measured until the host is fixed) · **Type:** service
validation (non-normative)
**Related:** [`adr/off-the-shelf-processing-service.md`](../adr/off-the-shelf-processing-service.md),
[`specs/processing.md`](../specs/processing.md#fr-15b-process--service),
[`reference/processing-service.md`](../reference/processing-service.md)

**Decision this supports:** which third-party service to adopt as the **first** processing
service under [adopt an off-the-shelf processing service](../adr/off-the-shelf-processing-service.md).
Pin down: does `ahmetoner/whisper-asr-webservice` (WhisperX) meet earshot's transcription +
whole-recording diarization needs, on a LAN host, at usable throughput, with a contract the
device can consume?

**Decision rule:** adopt if H1, H2, and H4 pass and H3 is acceptable on the target host;
otherwise evaluate another image against the same criteria — the
[ADR](../adr/off-the-shelf-processing-service.md) does not change.

## Assumptions

If one of these proves false, the decision reopens.

- **A1 — LAN host available:** the operator runs the service on a Docker host on the LAN,
  not on the Pi ([optional processing service](../adr/optional-processing-service.md)). If
  false, there is no service and the device transcribes locally.
- **A2 — gated models:** the pyannote diarization models require accepting their license on
  HuggingFace and a valid `HF_TOKEN`. If absent, diarization is unavailable but
  transcription still works (H2 cannot be evaluated).
- **A3 — device owns integration:** queuing, progress, retry, and speaker-label
  normalization are the device's job, so the service need not provide jobs, progress, or
  cancel ([off-the-shelf processing service](../adr/off-the-shelf-processing-service.md)).

## Hypotheses

- **H1 (transcription):** `POST /asr?output=json` returns time-stamped `{start, end, text}`
  segments for a speech clip.
- **H2 (diarization):** with `diarize=true`, every segment carries a `speaker`, distinct
  speakers are separated, and a label is consistent across the whole clip.
- **H3 (throughput):** processing runs at roughly **≥ 1× realtime** on the target host — so a
  43-minute session finishes in tens of minutes, far better than the Pi's sub-realtime local
  path (~7–13 min per 15 min).
- **H4 (contract fit):** capabilities are **discoverable** (the `diarize` parameter appears
  in `/openapi.json`) and the JSON response is **consumable** as `{start, end, text,
  speaker?}` after the device drops `words`/`word_segments`/`language`.

## Equipment

- Deployment under test: `ahmetoner/whisper-asr-webservice` **v1.9.1**,
  `ASR_ENGINE=whisperx`, at `http://10.1.0.36:9010`.
- **Stimulus (this run):** synthetic speech generated with `espeak-ng` at 16 kHz mono — a
  known sentence for H1/H3, and two distinct voices concatenated for a first-look at H2.
  Synthetic audio is a **limitation** (see Risks); a real 2-speaker `session.m4a` is needed
  to confirm H2 on representative capture.

## Instrumentation

- `curl` timing (`%{time_total}`) around each `POST /asr` for H3; capture HTTP status.
- Save the raw JSON response per request; assert the presence/shape of `segments[]` and,
  for diarize, `speaker` on each segment.
- `GET /openapi.json` for H4 capability discovery (already confirmed — see below).
- Hold constant: host, engine, `output=json`, `encode=true`.

## Data to collect

| Quantity | Unit | How | Decides | Measured (our run) |
| --- | --- | --- | --- | --- |
| HTTP status (transcribe) | — | `curl` | H1 | **500** (all attempts) |
| Segments returned | count | parse JSON | H1 | n/a — request 500s |
| Segment shape has `start/end/text` | bool | parse JSON | H1, H4 | n/a — request 500s |
| Clip duration | s | `ffprobe` | H3 baseline | 8.9 (espeak), 3.0 (sine) |
| Wall time (transcribe) | s | `curl %{time_total}` | H3 | ~0.5–0.7 (error, not processing) |
| Realtime factor | × | wall ÷ duration | H3 | n/a — no processing occurred |
| HTTP status (diarize) | — | `curl` | H2, A2 | not attempted (blocked by H1) |
| Every segment has `speaker` | bool | parse JSON | H2 | n/a |
| Distinct speakers found | count | parse JSON | H2 | n/a |
| `diarize` in `/openapi.json` | bool | `curl`+`jq` | H4 | ✅ confirmed |
| API/schema layer (`/docs`, `/openapi.json`) | — | `curl` | context | 200 (healthy) |

## Procedure

### Test A — transcription round-trip (H1, H3, H4-shape)
1. Synthesize a known sentence to 16 kHz mono WAV with `espeak-ng`.
2. `POST /asr?task=transcribe&output=json&encode=true` with the WAV as `audio_file`.
3. Record status, wall time, segment count/shape; compute realtime factor against clip
   duration.

### Test B — diarization round-trip (H2, A2)
1. Synthesize two distinct voices, each a different sentence; concatenate to one WAV.
2. `POST /asr?task=transcribe&diarize=true&output=json&encode=true`.
3. Record status; check every segment has `speaker` and how many distinct speakers appear.

### Test C — capability discovery (H4)
`GET /openapi.json`; confirm the `diarize` parameter on `POST /asr`.

## Success criteria

- **H1:** HTTP 200 with ≥ 1 segment carrying `start`, `end`, `text`.
- **H2:** HTTP 200 with a `speaker` on every segment and ≥ 2 distinct speakers on a
  genuine 2-speaker clip. (Synthetic voices are indicative only.)
- **H3:** realtime factor acceptable for the target host — near 1× on CPU is acceptable;
  report the measured figure, do not generalize it into a spec-level speed promise.
- **H4:** `diarize` present in `/openapi.json` **and** JSON segments consumable after
  discarding `words`/`word_segments`/`language`.

## Supporting evidence — prior verification (2026-07-25, from the handoff)

Not our run; recorded here as prior art that motivated adoption. A 90 s segment of a real
2-person meeting → `POST /asr?diarize=true&output=json` → HTTP 200, 18 segments, every
segment carried a `speaker`, two speakers separated (14 vs 4), turn boundary at ~76.7 s
accurate, `language: "en"`, wall time **~94 s** (≈ 1× realtime, CPU). Raw labels were
`SPEAKER_00`/`SPEAKER_01` by **cluster id** (`SPEAKER_01` spoke first) — the reason the
device normalizes to `Speaker N` by first appearance.

## Risks & confounders

- **Synthetic stimulus.** `espeak-ng` audio is clean and unlike ReSpeaker capture; it can
  validate the contract, response shape, and throughput, but **H2 (diarization quality) is
  not settled by synthetic voices** — a real 2-speaker `session.m4a` is required to confirm.
- **Throughput is host-specific.** The measured realtime factor is this host only; a GPU
  host is dramatically faster.
- **Diarization needs `HF_TOKEN`** on the host. If unset, Test B fails and only proves the
  degraded-but-not-broken path (transcription still works).

## Outcome

**Run 2026-07-25 (this session) — BLOCKED.** The deployment's API/schema layer is healthy
(`/docs`, `/openapi.json` → 200) and **H4 capability discovery passes** (`diarize` present in
the schema). But **every `POST /asr` and `POST /detect-language` returns HTTP 500** within
~0.5–0.7 s, identically for espeak-synthesized speech and an ffmpeg sine tone, across
`output=txt|json` and `encode=true|false`. A fast **500** (not a **400**) points to a
**server-side fault in the ASR pipeline** — most likely model load or WhisperX engine
configuration — not a request, format, or input problem.

**H1–H3 cannot be measured** and diarization (H2) was not attempted. Per the decision rule,
the service can be **neither adopted nor rejected** on this evidence: the fault is in *this
deployment*, not established as a property of the software. The prior 2026-07-25 verification
(above) shows the same image *can* work, so this is a regression/misconfiguration on the
host.

**Next steps (host-side, on `10.1.0.36`):**
1. `docker logs <container>` — read the traceback behind the 500 (this is the decisive datum
   and is not visible from the client).
2. Confirm `ASR_MODEL` weights are present/downloaded on the `cache` volume and that the
   WhisperX engine initializes (a fresh pull with no outbound access, or an OOM at model
   load, are common causes).
3. Once `POST /asr?output=json` returns 200, re-run **Test A** (H1/H3, this synthetic clip)
   and then **Test B** against a **real 2-speaker `session.m4a`** (H2) with `HF_TOKEN` set.

Until then the experiment stays **Running (blocked)**; the [ADR](../adr/off-the-shelf-processing-service.md)
decision to adopt an off-the-shelf service is unaffected — this is about validating the
specific deployment.
