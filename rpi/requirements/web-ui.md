# Web UI

The Raspberry Pi is headless with the LED as its only feedback channel. The web UI
gives users a screen — served by the application over the LAN and reached at the
Pi's IP address — to manage recordings and run/enrich processing without SSH.

It also becomes the **trigger** for transcription and diarization.

## Capabilities

| ID | Capability |
|---|---|
| FR-19 | The application serves a web UI over the LAN, reachable at the Pi's IP on a configured port. |
| FR-20 | List all sessions with derived status (recording / pending / transcribed / diarized / failed), timestamp, duration, and size. |
| FR-21 | Play and download a session's `session.wav` in the browser. |
| FR-22 | Delete a session (removes its directory; frees disk — see [storage.md](../specs/storage.md#disk-space-management)). Display confirmation. |
| FR-23 | Start and stop a recording from the web UI, shared with the hardware button. |
| FR-24 | Initiate transcription on demand for a pending session, and show progress, failures, and retry. |
| FR-25 | Diarize a session — available only when OpenAI is configured (see [Diarization](#diarization)). |
| FR-26 | Enter, update, and clear the OpenAI API key; a valid key gates FR-25. |
| FR-27 | Enroll named speakers: capture and name short reference clips (2–10 s, up to 4) so diarized transcripts use real names instead of `Speaker N`, and so identity stays consistent across a split long session. |

## Settled decisions

- **Access model (v1): trusted LAN, no login.** The web UI binds to the LAN and has
  no authentication; anyone who can reach the Pi's IP can use it. Acceptable for a
  home/trusted-network device in v1.
- **The OpenAI API key lives in `config.toml`** (`[diarization].api_key`) and can be
  set from either the installer/config file or the web UI (FR-26). Since `config.toml`
  now holds a secret, keep it out of version control and restrict its permissions.
- **Local transcription is the default and the only offline path.** Local
  faster-whisper (`base.en`) produces `transcript.md` with no key and no network
  (FR-24). A configured OpenAI key adds the optional **diarize** action (FR-25);
  un-diarized sessions stay fully offline and only an explicit diarize call uses the
  network.
- **Recording control is shared** (FR-23). The button and the web UI both start/stop
  the single active session; the LED and state machine reflect the same state
  regardless of which control acted.
- **One transcript per session; diarization replaces it** (TD-5, resolved). Every
  session has a single `transcript.md`. Local faster-whisper writes it by default;
  running diarize **overwrites** `transcript.md` with the speaker-labelled version
  (the OpenAI model transcribes and labels in one pass, so its output *is* the
  transcript). No separate diarized file is kept.

## Diarization

When an OpenAI key is configured, a session can be diarized via OpenAI's
**`gpt-4o-transcribe-diarize`** model, which transcribes *and* labels speakers in a
single `/v1/audio/transcriptions` call (`diarized_json`: per-segment speaker labels +
timestamps). Because the model produces a full transcript in the same pass, the diarized
result **replaces** the session's `transcript.md` rather than sitting beside it — so
there is never a second, divergent transcript of the same audio.

- **Output:** the diarized result **overwrites** `transcript.md`; the session's single
  transcript becomes the speaker-labelled one (TD-5). The raw OpenAI response is saved to
  `transcript_diarized_raw.json`, whose presence marks the current `transcript.md` as the
  diarized version (FR-20 status). Diarization does not require a prior local transcript —
  it can produce `transcript.md` on its own.
- **Named speakers:** the UI enrolls speakers (FR-27) and passes their clips as
  `known_speaker_names[]` / `known_speaker_references[]`, so the transcript shows real
  names instead of `Speaker N`. Without enrollment, generic `Speaker N` labels are used.
- **Requires network + a valid key;** without either the diarize action is unavailable.
  On failure the existing `transcript.md` (if any) is left intact — the overwrite happens
  only on success.
- **Long sessions** exceed OpenAI's per-request limits (25 MB / 1500 s) and must be
  compressed and/or split, reusing the enrolled references to keep speaker identity
  consistent across parts. Mechanism and validation: **TD-7**.
- **Quality** is still bounded by the mono, closely-spaced-mic capture; real-audio
  validation is part of TD-7.

Reference: [gpt-4o-transcribe-diarize model doc](https://developers.openai.com/api/docs/models/gpt-4o-transcribe-diarize).

## Out of scope (v1)

- Authentication / user accounts on the web UI.
- Summarization (a designed-for future action; not built).
- Editing `config.toml` from the UI **beyond the OpenAI key** (FR-26) — other settings
  remain an SSH + `systemctl restart` operation.

## Where the behavior is specified

This requirement is threaded into the specs/docs below:

- `specs/transcription.md` — web-initiated trigger and the diarization path.
- `specs/state-machine.md` — web-initiated start/stop (FR-23) and the Transcribing state.
- `specs/configuration.md` — `[web]` and `[diarization]` settings.
- `specs/storage.md` — the single `transcript.md` and the `transcript_diarized_raw.json` marker.
- `requirements/connectivity.md` / `non-functional.md` — LAN web UI, internet only for
  diarization.
- `requirements/backlog.md` — B-I1 promoted; summarization tracked as B-T5.

The exact HTTP endpoints and payloads still await a dedicated `specs/web-ui.md`
(pending TD-7 validation).

## Open decisions

TD-5 (single `transcript.md`; diarization overwrites it) and TD-6 (named-speaker
enrollment) are resolved and folded in above. Remaining: **TD-7** (long-session audio
upload to OpenAI) in
[open-technical-decisions.md](open-technical-decisions.md).
