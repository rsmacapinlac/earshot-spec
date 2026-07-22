# Web UI

The Raspberry Pi is headless with the LED as its only feedback channel. The web UI
gives users a screen — served by the application over the LAN and reached at the
Pi's IP address — to manage recordings and run/enrich processing without SSH.

It also becomes the **trigger** for transcription and diarization.

## Capabilities

| ID | Capability |
|---|---|
| FR-19 | The application serves a web UI over the LAN, reachable at the Pi's IP on a configured port. |
| FR-20 | List all sessions with derived status (recording / pending / transcribed / diarized / failed), identity, duration, and size. A session is identified by its **name** (FR-29) when set, otherwise by its session ID; the ID is always available as the secondary identifier. Display wording: pending renders as **"Audio only"**, diarized as **"Transcribed with Speakers"**. |
| FR-21 | Play and download a session's `session.m4a` in the browser. |
| FR-22 | Delete a session (removes its directory; frees disk — see [storage.md](../specs/storage.md#disk-space-management)). Display confirmation. |
| FR-23 | Start and stop a recording from the web UI, shared with the hardware button. |
| FR-24 | Initiate transcription on demand for a pending session, and show stage, progress, failures, and retry. **Always available** — it runs locally unless a processing service is configured. |
| FR-25 | Diarize a session — offered only when a processing service is configured and reports diarization available, and when no other job is in flight (see [Diarization](#diarization)). |
| FR-27 | **Name the speakers** in a diarized session: for each detected `Speaker N`, play a sample clip of that voice drawn from the session and assign a name. Names replace the `Speaker N` labels throughout that session. Naming is **per session**, applied **after** diarization, and requires no set-up beforehand. A session split across multiple requests lists each part's labels separately, so the same person may need naming more than once. |
| FR-28 | Show **device status** — the live state the on-device LED reflects (booting / ready / recording / finalizing / processing / disk threshold), continuously and on every view. |
| FR-30 | **Configure an optional processing service** — set, update, and clear `processing.service_url`, and show live connection status: unset, unreachable, or reachable with the capabilities it reports. Unset is a normal state, presented as such and never as an error. Applies without a restart. |
| FR-29 | **Name and rename a session** from the web UI. The name is optional, free-text, need not be unique, and can be set or changed at any point in a session's life — including while it is still recording. It is shown wherever the session is identified, is used for the transcript header and the audio download filename, and is cleared by emptying it. |

## Settled decisions

- **Access model (v1): trusted LAN, no login.** The web UI binds to the LAN and has
  no authentication; anyone who can reach the Pi's IP can use it. Acceptable for a
  home/trusted-network device in v1.
- **The device stands alone; a service is an upgrade.** Transcription always works —
  locally by default, or on a [processing service](../../service/README.md) if one is
  configured ([ADR-0010](../adr/0010-optional-processing-service.md)). A fresh Pi with no
  configuration transcribes. Adding a service makes that far faster and unlocks
  diarization; removing it falls back, and nothing is lost.
- **Diarization is the one capability that needs a service.** It requires compute a Pi 4B
  does not have. Without a service the action is simply not offered — presented as a
  capability this deployment lacks, not as a failure.
- **The UI offers only what is actually available.** Capabilities come from the service's
  health endpoint, so a deployment with transcription but no diarization models shows one
  action, not two that fail differently (FR-30).
- **No API keys and no third parties.** There is no cloud path for anything — FR-26 stays
  retired.
- **Recording control is shared** (FR-23). The button and the web UI both start/stop
  the single active session; the LED and state machine reflect the same state
  regardless of which control acted.
- **The web UI is parity for recording, and an extension for processing.** Record,
  stop, and status are the same capability on two surfaces (button + LED, and the web
  UI). Transcription (FR-24) and diarization (FR-25) exist **only** on the web surface —
  the button has no gesture for them. As extensions they must never degrade recording,
  the base function: at most one processing job runs at a time, and any that contends
  with capture for CPU yields to it — in practice only local transcription does. Rules:
  [`specs/processing.md`](../specs/processing.md#processing-jobs).
- **One transcript per session; diarization replaces it** (TD-5, resolved). Every session
  has a single `transcript.md`, rendered by the device from the service's segments. A
  transcribe job writes it; a diarize job **overwrites** it with the speaker-labelled
  version, since the service labels and transcribes in one job so its output *is* the
  transcript. No separate diarized file is kept.
- **Speakers are named after diarization, per session — not enrolled beforehand**
  (TD-6, re-decided). Diarization always returns generic `Speaker N`; the user then
  plays a sample clip of each voice and names it, and the names replace the labels in
  that session's `transcript.md`. There is no enrollment step, no cross-session speaker
  registry, and no identity carried between recordings — the same person named in two
  sessions is named twice, independently. Chosen because it needs no set-up before a
  meeting and lets the user name each voice while reading what it actually said. The
  cross-part identity problem this leaves open is handled *by* this naming step: a split
  session lists each part's labels and the user names them, which reconciles identity
  without any cross-request trickery (TD-7, resolved).
- **Sessions are identified by name, not by date** (FR-29). Identity is the allocated
  session ID `rec-NNNNNN`, which contains no date
  ([ADR-0008](../adr/0008-session-identity.md)). On top of it the user can put a
  free-text name, which is what the UI leads with and what travels into the transcript
  header and the download filename. An unnamed session falls back to its ID. The capture
  date is shown as a scanning convenience only, never as identity, and nothing in the
  interface requires the clock to have been right. This matches the ESP32 track, where
  `rec-NNNNNN` is identity and the spoken label is the human handle.
- **Status is the third parity item** (FR-28). The web UI continuously shows the state
  the LED is showing, on every view — so a user can never stop a recording from a page
  that never told them one was running, and the disk-threshold block stops being
  LED-only on a headless device.

## Diarization

A session can be **diarized** when the processing service reports the capability: the
service transcribes and labels speakers in one job, returning segments tagged
`Speaker 1`, `Speaker 2`, … The result **replaces** the session's `transcript.md` rather
than sitting beside it, so there is never a second, divergent transcript of the same
audio.

- **Output:** the diarized result **overwrites** `transcript.md` (TD-5). The raw
  speaker-labelled segments are saved to `transcript_diarized_raw.json`, whose presence
  marks the current transcript as the diarized version (FR-20 status). Diarizing does not
  require a prior transcript — it produces one.
- **Named speakers:** the service always returns generic `Speaker N`. The UI offers a
  sample clip of each detected voice — drawn from that speaker's own segments — and the
  user names them (FR-27). Names are substituted into `transcript.md`; the mapping
  persists in `session.json` so it survives a reload and can be corrected later.
- **Length is not a special case.** The service diarizes the whole recording in one pass,
  so a speaker keeps one label from beginning to end however long the session is. There is
  no split, no per-part relabelling, and no session length at which the behaviour changes.
- **Availability.** Requires a configured service reporting `diarize: true`, and no other
  job in flight. Without one the action is not offered at all. On failure the existing
  `transcript.md`, if any, is left intact — the overwrite happens only on success.
- **It never contends with recording**, because the work happens on another machine. The
  LED shows **Processing** when a job runs alone and **Recording** when both are active;
  the web UI names what is running. (A *local* transcription does contend, and is
  cancelled by a recording — see
  [`specs/processing.md`](../specs/processing.md#preemption).)
- **Quality** is bounded by the mono, closely-spaced-mic capture, and is not gated by a
  decision — with the models on your own hardware it costs nothing per attempt, so the user
  judges it on their own audio.
- **Reverting** a diarized session is always possible, even after the service is gone: a
  local re-transcribe removes the speaker labels.

Behavior contract: [`specs/processing.md`](../specs/processing.md#diarization-fr-25).
Service side: [`service/specs/processing.md`](../../service/specs/processing.md#diarize).

## Out of scope (v1)

- Authentication / user accounts on the web UI.
- **Speaker enrollment and any cross-session speaker registry** (TD-6, re-decided).
  Names are assigned per session, after the fact.
- Summarization (a designed-for future action; not built).
- Editing `config.toml` from the UI **beyond the processing service URL** (FR-30) —
  other settings remain an SSH + `systemctl restart` operation.
- Deploying or managing a processing service. The UI points at one if you have it;
  standing it up is a separate operation
  ([service deployment](../../service/specs/deployment.md)).

## Where the behavior is specified

This requirement is threaded into the specs/docs below:

- `specs/processing.md` — web-initiated trigger, job submission and polling, the
  diarization path, and the one-job-at-a-time rule.
- `specs/state-machine.md` — web-initiated start/stop (FR-23) and the Processing state.
- `specs/configuration.md` — `[web]` and `[processing]` settings.
- `specs/storage.md` — the single `transcript.md`, the `transcript_diarized_raw.json`
  marker, and the per-session `session.json` holding user-authored labels (FR-27, FR-29).
- `requirements/connectivity.md` / `non-functional.md` — LAN-only; no internet required.
- `requirements/backlog.md` — B-I1 promoted; B-T6 promoted as an optional capability;
  summarization tracked as B-T5.
- `../service/` — the optional processing service this UI can drive.

The exact HTTP endpoints and payloads still await a dedicated `specs/web-ui.md`.

## Open decisions

TD-5 (single `transcript.md`; diarization overwrites it), TD-6 (speaker naming — now
**post-hoc and per-session**, no enrollment) and TD-7 (long sessions are **split without
cross-request label stitching**) are all resolved and folded in above. The
[technical-decisions registry](open-technical-decisions.md) is empty.
