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
- **Transcription stays on-device.** Local faster-whisper (`base.en`) remains the
  transcription engine (FR-24). The OpenAI key unlocks **diarization only**; it does
  not move transcription to the cloud. The base product stays fully offline; only
  diarization needs network.
- **Recording control is shared** (FR-23). The button and the web UI both start/stop
  the single active session; the LED and state machine reflect the same state
  regardless of which control acted.
- **Diarization output is two separate artifacts** (TD-5, resolved): the local plain
  `transcript.md` and a `diarized.md`.

## Diarization

When an OpenAI key is configured, a session can be diarized via OpenAI's
**`gpt-4o-transcribe-diarize`** model, which transcribes *and* labels speakers in a
single `/v1/audio/transcriptions` call (`diarized_json`: per-segment speaker labels +
timestamps). This is genuine acoustic diarization, produced independently of the local
plain transcript.

- **Output:** a separate `transcript_diarized.md`, alongside the untouched local
  `transcript.md` (TD-5).
- **Named speakers:** the UI enrolls speakers (FR-27) and passes their clips as
  `known_speaker_names[]` / `known_speaker_references[]`, so the transcript shows real
  names instead of `Speaker N`. Without enrollment, generic `Speaker N` labels are used.
- **Requires network + a valid key;** without either the diarize action is unavailable
  and transcription (FR-24) is unaffected.
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
- `specs/storage.md` — the `transcript_diarized.md` artifact.
- `requirements/connectivity.md` / `non-functional.md` — LAN web UI, internet only for
  diarization.
- `requirements/backlog.md` — B-I1 promoted; summarization tracked as B-T5.

The exact HTTP endpoints and payloads still await a dedicated `specs/web-ui.md`
(pending TD-7 validation).

## Open decisions

TD-5 (two-artifact output) and TD-6 (named-speaker enrollment) are resolved and folded
in above. Remaining: **TD-7** (long-session audio upload to OpenAI) in
[open-technical-decisions.md](open-technical-decisions.md).
