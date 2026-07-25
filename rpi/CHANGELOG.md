# Changelog

All notable changes to the earshot **Raspberry Pi documentation** (specs,
requirements, ADRs, reference, experiments). The documentation set is versioned as
a whole; the current version is recorded in `../AGENTS.md`. Dates are ISO-8601
(YYYY-MM-DD).

## [Unreleased]

_Nothing yet._

## [1.2] — 2026-07-25

### Added
- **Optional per-session date/time** (`requirements/web-ui/set-session-datetime.md` — new;
  `specs/api.md`, `specs/storage.md`, `specs/processing.md`,
  `requirements/web-ui/list-sessions.md`,
  `requirements/non-functional/clock-independence.md`). A user may optionally assert when a
  conversation happened, stored as **`occurred_at`** — set, changed, or cleared via
  `PATCH /v1/sessions/{id}` exactly like the name, with date and time each optional. It is
  distinct from the clock-derived `created_at`: a trustworthy user-provided label that shows
  in the session list and as a `**Date:**` line in the `transcript.md` header (rewritten in
  place on change), yet stays **descriptive-only** — nothing sorts, looks up, or establishes
  identity by it. Clock-independence is refined to permit a *user-set* date in the header
  while still forbidding a *clock-derived* one.

## [1.1] — 2026-07-25

### Added
- **The processing service is now documented in this track** as an adopted off-the-shelf
  dependency, replacing the retired bespoke `service/` track:
  [off-the-shelf processing service](adr/off-the-shelf-processing-service.md) (ADR — adopt an
  existing third-party service rather than build one),
  [experiment 0002](experiments/0002-whisper-asr-webservice.md) (validates
  `ahmetoner/whisper-asr-webservice` / WhisperX), and
  [reference/processing-service.md](reference/processing-service.md) (deployment recipe).

### Changed
- **Transcription/diarization routes to the adopted synchronous service**
  (`specs/processing.md`, `specs/api.md`, `specs/configuration.md`,
  `requirements/web-ui/transcribe.md`, `requirements/web-ui/processing-service.md`). The
  contract is the service's own single blocking `POST /asr`, not a bespoke async job API. The
  service worker submits one request and blocks for the result instead of submit/poll: the
  `jobs.remote_job_id` column is dropped, an interrupted service job is **re-run** rather than
  resumed by polling, and `poll_interval_seconds` is replaced by `request_timeout_seconds`. A
  service job carries **no stage/progress** (synchronous and opaque); local transcription
  still reports progress. **Speaker labels are normalized on the device** — raw WhisperX
  `SPEAKER_NN` → `Speaker N` by first appearance. Service **capabilities** are discovered by
  probing `/openapi.json` (there is no health endpoint). `POST /v1/sessions/{id}/jobs` gains
  an optional `num_speakers` hint (mapped to `min_speakers`/`max_speakers`). The device's
  browser-facing HTTP API is otherwise unchanged, so the web UI needs no structural change.
- **Superseded-ADR convention adopted** (`../AGENTS.md`, `adr/README.md`): a wholesale-replaced
  ADR moves to `adr/superseded/` with a pointer to its replacement. The three former
  `service/` ADRs (`separate-processing-service`, `open-source-diarization`, `async-job-api`)
  now live in [`adr/superseded/`](adr/superseded/README.md).
- **Processing is never auto-triggered.** Finalizing a recording does not enqueue a job;
  a stopped recording becomes pending and waits for the user to initiate transcription or
  diarization from the web UI. Resolves the open trigger question and fixes a stale
  "queue the session for transcription" step in `specs/recording.md`
  (`specs/processing.md`, `specs/recording.md`).
- **The HTTP API is the device's operating interface** (`adr/http-api-is-the-interface.md`
  — new; `requirements/non-functional/core-functionality-over-api.md` — new;
  `specs/api.md` — new). The app exposes one API and the web UI is a client of it, with no
  privileged path; every core operation (recording, browsing, playback, download, delete,
  transcription, diarization, naming, status) is reachable over HTTP, and the on-disk
  layout is an implementation detail rather than a public contract. **Configuring** the
  device stays an SSH edit of `config.toml` — deliberately out of the API, since it is
  administration done rarely, not a core operation. `specs/api.md` drafts the endpoints,
  including the two capabilities that were previously uncaptured: transcript export
  (`GET …/transcript`) and live state reflection (an SSE `/v1/events` stream).
- **`earshot-tui` file coupling reduced.** Dropped the tui-specific justifications from the
  FR-n convention note, the transcript-format label, `status.json`'s "export" role, and the
  identity examples — consistent with the on-disk layout no longer being a public contract.
  The `"encoded"` status literal keeps its tui note for now, pending a separate decision.
- **State moves to SQLite; files hold only artifacts** (`adr/state-storage.md`, replacing
  the filesystem-as-state decision; `specs/storage.md`, `specs/processing.md`,
  `specs/configuration.md`). Session records, the speaker-name map, and the processing
  queue live in `earshot.db` (WAL mode, stdlib `sqlite3` — no daemon, no dependency).
  Audio, transcripts, and a rebuildable `status.json` stay as files. The database is
  reconstructable from those files, so losing it costs nothing permanent. Retires the
  per-session state files (`session.json`, `.job.json`, `.processing_failures.json`,
  `.failed_processing`, `.next_id`).
- **Session identity is the database primary key** (`adr/session-identity.md`, amended).
  Allocation is `INTEGER PRIMARY KEY AUTOINCREMENT`, so IDs are monotonic and never
  reused. Supersedes the filesystem `max + 1` scan, which reused an ID after the newest
  session was deleted.
- **A real processing queue** (`specs/processing.md`): the `jobs` table — ordered,
  durable, retryable, inspectable — replaces a queue inferred by scanning for files.
  Route (local vs service) is chosen at dequeue; crash resilience is per-route.
- **Job execution: in-process worker, no task-queue framework** (`adr/job-execution.md` —
  new). One worker thread over the table; local inference runs in a subprocess so an OOM
  kill costs the job, not the recording, and cancellation is a signal rather than a
  cooperative check. Celery/RQ/Dramatiq/arq rejected for requiring a broker daemon.
- **Data directory split from the install directory.** `storage.data_dir` defaults to
  `~/earshot-data` and holds `config.toml`, `earshot.db`, and `recordings/`; `~/earshot`
  is left as a pure git checkout whose update path is `git pull`. `ReadWritePaths` follows
  the data directory (`specs/configuration.md`, `specs/install-service.md`).
- **Clock independence promoted to a requirement**
  (`requirements/non-functional/clock-independence.md` — new). Identity, ordering and
  labelling must never depend on the clock; the session-identity ADR now satisfies it
  rather than re-deriving it.

### Docs structure
- **`requirements/non-functional.md` → `requirements/non-functional/`** — a README index
  plus one file per requirement (no internet, clock independence, resilience, startup),
  referred to by name rather than `NFR-n`.
- **`requirements/web-ui.md` → `requirements/web-ui/`** — a README plus one file per
  capability. `FR-19`–`FR-30` retired as identifiers; the web UI capabilities are now
  named rather than numbered.
- **ADRs de-numbered** in the `rpi/` track and the then-existing bespoke `service/` track —
  files, headings, and cross-references now use names (`adr/state-storage.md`, not
  `0006-…`), matching the requirements folders. `esp32/` is unchanged.
- **`requirements/backlog.md` removed** — its live items (real-time transcription,
  summarization) survive as out-of-scope entries; B-T6 was delivered.

### Added
- **Web UI requirement** (`requirements/web-ui.md`): a LAN-served web UI (FR-19–FR-29)
  to browse / listen / delete recordings, start/stop recording, name sessions, show device
  status, initiate transcription, and — with an OpenAI key — diarize sessions via
  `gpt-4o-transcribe-diarize` and name the detected speakers. Trusted-LAN / no-auth in v1;
  the OpenAI key lives in `config.toml` (`[diarization].api_key`) and is settable from the
  web UI.
- **Experiment 0001** (`experiments/0001-storage-bitrate.md`): validates the stored
  AAC bitrate for `session.m4a` against a PCM control — transcription WER, listen-back,
  and encode wall-clock (supports the audio storage format decision). Replaces an earlier 0001 that planned to
  validate OpenAI diarization quality and cross-part stitching; that experiment never ran
  and its decision (TD-7) was closed by decision instead.
- **Capture front-end spec** (`specs/recording.md`): the WM8960 is configured for
  ALC using Wolfson's speech preset (target −12 dBFS, fast 24 ms attack / 384 ms
  decay, noise gate + HPF on, Max Gain capped at 5 provisionally, on the captured
  left channel), with the `amixer` values and the requirement to persist them to
  `/etc/voicecard/wm8960_asound.state`.

### Changed
- **An optional processing service; the device still stands alone**
  (`adr/optional-processing-service.md` — new; `specs/processing.md` — renamed from
  `transcription.md`; `specs/configuration.md`, `specs/state-machine.md`,
  `specs/storage.md`, `specs/install-service.md`, `requirements/web-ui.md`,
  `requirements/non-functional.md`, `requirements/connectivity.md`). Transcription runs
  **locally by default** — a fresh Pi transcribes with no service, account, or internet.
  Setting `processing.service_url` routes it to an
  [earshot processing service](reference/processing-service.md) on the LAN instead, which is far
  faster and is the **only** way to enable diarization; clearing it falls back and nothing
  is lost. A required service was drafted and reversed: a recorder that obliges you to
  stand up and maintain a container is a different product from one you plug in.
- **Diarization has no local path and no cloud path.** It needs compute a Pi 4B lacks, so
  it requires a service or is not offered — stated plainly rather than papered over with a
  third-party API. The OpenAI path is gone: no key, no cost, no 25-minute cap, no duplicate
  speaker labels, and no internet dependency anywhere. **FR-26 retired**, **FR-30 added**
  for optional service configuration and live capability status.
- **Preemption survives, scoped to where the CPU is.** Local transcription yields to
  recording; a service job does not, because the work is on another machine. One rule —
  *whatever holds CPU on the Pi yields to capture* — with the route deciding whether
  anything does (`specs/processing.md`, `specs/state-machine.md`).
- **TD-7 dissolved and B-T6 promoted.** A service diarizes a whole recording in one pass,
  so the per-request cap, the split, and the duplicate `Speaker N` entries have no subject
  left (`requirements/open-technical-decisions.md`, `requirements/backlog.md`).
- **NFR-1 rewritten — standalone, and no internet dependency.** Recording, storage,
  playback, the web UI *and transcription* all work on the device alone; nothing anywhere
  requires the internet (`requirements/non-functional.md`, `requirements/connectivity.md`).
- **The v1.0 design persisted service job state.** `.job.json` in the session directory held
  the service's `job_id`, so a reboot mid-job resumed polling instead of resubmitting work
  already done remotely. A local job needed no such record.
  `.transcription_failures.json` / `.failed_transcription` are renamed
  `.processing_failures.json` / `.failed_processing`. An unreachable service is reported as
  a connection problem and does **not** burn per-session retries — transcription can fall
  back to local in the meantime (`specs/storage.md`, `specs/processing.md`).
- **`[hardware]` section added to the config schema** — `hardware.hat` was referenced by
  the installer, `hardware.md`, and the hardware-abstraction-layer ADR but missing from the authoritative schema
  (`specs/configuration.md`).
- **TD-6 re-decided — speaker naming is post-hoc and per-session; enrollment is out of
  scope.** Diarization always returns generic `Speaker N`; the user then plays a sample
  clip of each detected voice and names it, and the names are substituted into that
  session's `transcript.md`. The map persists in the per-session `session.json`. No
  enrollment step, no cross-session speaker registry, no identity carried between
  recordings. FR-27 rewritten (`requirements/web-ui.md`, `specs/transcription.md`,
  `specs/storage.md`). **Consequence:** enrollment was the user-facing source of the
  reference clips that hold identity across a split request — which is what led to TD-7
  being closed by decision rather than by validation (below).
- **Sessions are stored compressed — `session.m4a`, AAC-LC 32 kbps** (`adr/audio-storage-format.md`
  re-decided; `specs/recording.md`, `specs/storage.md`, `specs/transcription.md`,
  `specs/configuration.md`, `specs/state-machine.md`). Capture is unchanged and still
  writes lossless PCM chunks for crash resilience; finalization now concatenates **and
  encodes** them in a single `ffmpeg` pass (concat demuxer → AAC), writing no
  intermediate full-length WAV. A 43-minute session drops from ~83 MB to ~10 MB, so a
  59 GB card holds ~230 hours instead of ~28. Bitrate is tunable via
  `recording.encode_bitrate_kbps`; container and codec are fixed. That ADR was re-decided
  in place rather than superseded — nothing had been built against the WAV format, and the
  file is named by topic, so every existing link still resolves.
- **Diarization no longer transcodes, and the two-phase preemption rule collapses.**
  The stored file is already an accepted upload format, so a ≤25-min session uploads
  as-is and a longer one is split with a stream copy. With no CPU-bound phase left,
  diarization never contends with capture: it is never preempted and may be started
  during a recording. Only local transcription still yields
  (`specs/transcription.md`, `requirements/web-ui.md`).
- **`diarization.upload_format` removed** — there is nothing left to transcode
  (`specs/configuration.md`). TD-7's compression step is likewise gone; its "Earshot
  math" now shows a full 25-minute request at ~6 MB, so size never binds
  (`requirements/open-technical-decisions.md`).
- **The `"encoded"` status literal became truthful.** It was retained for `earshot-tui`
  compatibility with a note that no encode occurred; one now does.
- **Backlog B-T6 — open-source / self-hosted diarization** (`requirements/backlog.md`).
  Logged as the alternative that would remove the constraint behind TD-7 rather than work
  around it: the 25-min split, the API key, the per-session cost, and NFR-1's single
  network dependency all follow from the chosen provider, not from the problem. Notes the
  reframing that makes it tractable — earshot already transcribes locally, so it needs
  only speaker turns, not a transcribe-and-diarize service — plus candidates
  (pyannote.audio, WhisperX, sherpa-onnx) and the Pi 4B capacity caveat.
- **TD-7 resolved — long sessions are split, not stitched.** Sessions over OpenAI's
  25-minute per-request limit are split into ≤25-min parts and diarized independently,
  with **no attempt to correlate speaker labels across parts**. The only mechanism for
  that was `known_speaker_references`, an undocumented workaround that TD-6 left with no
  vetted source of clips. Post-hoc naming (FR-27) already reconciles identity: the UI
  lists every part's labels and the user names them, so a 43-minute two-person meeting is
  four names instead of two. This removes the undocumented dependency, the auto-carving
  step, and its unspecified clip-selection rule
  (`requirements/open-technical-decisions.md`, `requirements/web-ui.md`,
  `specs/transcription.md`). The technical-decisions registry is now empty.
- **Diarization quality is not a release gate.** It is opt-in, off by default, requires a
  user-supplied key, and is reversible by re-transcribing, so the user judges it on their
  own audio rather than it blocking v1 (`requirements/web-ui.md`).
- **Storage bitrate flagged provisional.** 32 kbps ships, but is to be confirmed during
  bring-up — the same treatment as `ALC Max Gain` — because the encode is one-way
  (`specs/recording.md`, `specs/configuration.md`).
- **Session identity is now an allocated `rec-NNNNNN` ID, not a timestamp**
  (`adr/session-identity.md` — new; `specs/storage.md`, `specs/recording.md`,
  `specs/transcription.md`, `requirements/web-ui.md`). The Pi 4B has no RTC and NFR-1
  requires offline recording, so a cold boot without a network named sessions from a
  clock known to be wrong — mis-ordering the FIFO queue and propagating a wrong date into
  the transcript. Since the button-press path has no time source at all, the dependency
  was removed rather than supplied: IDs are allocated `max + 1` over existing `rec-*`
  directories, with an optional non-authoritative `.next_id` hint. Queue order is now
  true capture order unconditionally. Matches the ESP32 track's identity model.
- **Capture time demoted to metadata.** `status.json` keeps `recorded_at` as descriptive
  information only — nothing sorts, looks up, or recovers by it, and it stays a true
  mirror because it is re-derivable from the session directory's creation time if lost or
  rebuilt. That same fallback covers crash recovery, where the in-memory start time is
  gone (`specs/storage.md`).
- **Session naming (FR-29).** A session can be given an optional free-text name from the
  web UI at any point, including mid-recording; the UI leads with it and falls back to the
  session ID. `transcript.md` **drops the recording timestamp from its header** in favour
  of the name (or the ID when unnamed), plus a `**Session:**` line carrying the ID;
  renaming rewrites that header in place. `**Processed:**` is the only clock-derived field
  left in the transcript, and is descriptive only.
- **`speakers.json` consolidated into `session.json`.** One per-session file now holds
  everything the user authored — the session name and the `Speaker N` → name map —
  keeping user-authored labels out of the rebuildable `status.json` mirror, which the
  app is entitled to regenerate. Both keys optional; an absent or unparseable file means
  "unnamed" and is never an error (`specs/storage.md`).
- **Device status added as a parity item (FR-28).** The web UI continuously shows the
  state the LED reflects (booting / ready / recording / finalizing / processing / disk
  threshold). Record, stop, *and status* are now all parity capabilities across the two
  surfaces; the disk-threshold condition is no longer LED-only on a headless device.
- **FR-26: the OpenAI key is write-only.** The stored value is never returned to the
  browser or pre-filled into the field — only a masked fingerprint is shown.
- **FR-20 display wording:** pending renders as "Audio only", diarized as "Transcribed
  with Speakers".
- **`.failed_transcription` is cleared by the UI's Retry action** (FR-24), not only by
  manual deletion (`specs/storage.md`).
- **Transcription is web-initiated** from the web UI, on demand
  (`specs/transcription.md`, `specs/state-machine.md`).
- **Recording control is now shared** between the button and the web UI (FR-23).
- **Diarization added** via OpenAI `gpt-4o-transcribe-diarize`, gated on a configured
  key. Each session has a **single `transcript.md`**: local faster-whisper writes it by
  default, and a diarize run **overwrites** it with the speaker-labelled version (the raw
  OpenAI response is saved to `transcript_diarized_raw.json`, whose presence marks the
  diarized state). No separate diarized transcript file (`specs/transcription.md`,
  `specs/storage.md`).
- **Network framing qualified.** The app now serves a LAN web UI, and diarization needs
  internet; `requirements/connectivity.md` and `non-functional.md` (NFR-1) updated —
  recording and local transcription stay offline.
- **Config additions** (`specs/configuration.md`): `[web]` and `[diarization]` sections.
- **New technical decisions:** TD-5 (**single `transcript.md`; diarization overwrites it
  in place**) and TD-6 (named-speaker enrollment) resolved into `web-ui.md`; TD-7
  (long-session upload to OpenAI) subsequently resolved as split-without-stitching.
- **Backlog:** B-I1 (Web UI) promoted into the active requirements; B-T5
  (summarization) added as a designed-for future item.
- **TD-1 resolved and removed.** The capture-gain question (fixed PGA vs. ALC) is
  decided in favour of ALC and folded into `specs/recording.md`; the
  `reference/` front-end section now points to the spec. `ALC Max Gain` ships as a
  starting value (5) to confirm on hardware during bring-up.
- **TD-2 resolved and removed.** Capture is now **mono** (the left mic), not
  stereo: faster-whisper downmixes to mono anyway, the closely-spaced mics carry
  no usable stereo image, and mono halves the WAV size. A single channel is taken
  (not an L+R average) to avoid comb filtering on off-axis talkers. `config.toml`
  default `audio.channels` becomes `1`; `recording.md`/`configuration.md`/
  `storage.md` updated.
- **TD-3 resolved and removed — Pi 4B is the minimum; Pi Zero 2W is out of scope.**
  Dropped the Pi Zero 2W removable-device design-intent content from `hardware.md`,
  `transcription.md`, and the systemd capability rationale
  (`CAP_SYS_MODULE`/`CAP_SYS_ADMIN` no longer needed).
- **TD-4 resolved and removed — default transcription model is now `base.en`**
  (was `tiny.en`), for better accuracy on the Pi 4B; `tiny.en` stays available as
  the lighter alternative. `configuration.md`/`transcription.md` updated. The
  technical-decisions registry is now empty.
- **UX-1/UX-2/UX-3 resolved and removed** — all kept the v1 behavior already in
  the specs: LED colour overload accepted (single LED, disambiguated by pulse
  speed); audio feedback is out of scope; and the single-button gestures are
  unchanged (3 s idle hold = shutdown, no confirmation — low-stakes since captures
  commit first). The UX registry is now empty.
- **Removed all speaker / audio-output content** — the hardware isn't present.
  Dropped backlog B-A1/B-A2 (audio cues + volume), FR-5, the `AudioOutputInterface`
  mention (hardware abstraction layer ADR), and the Speaker rows from `hardware.md`
  and `reference/`.
  The LED is the sole feedback channel.
- **Backlog triaged.** Dropped B-T2 (installer model prompt) and B-I2 (Pi 5
  support). **B-I1 (Web UI / dashboard) promoted to the next release.**
- **Retired FR-11 from v1.** Dropped the related state-machine path,
  dependencies, and transcription cancellation path.

### Fixed
- **Documentation audit corrections** across the RPi set:
  - **Python/OS mismatch.** The target OS is Debian 13 "trixie" (Python 3.13);
    the docs called for a "Python 3.11 venv" as if 3.11 shipped with it. Clarified
    that 3.11 is the *minimum* and the venv uses the newer OS default
    (`adr/python-venv-over-docker.md`, `specs/install-service.md`).
  - **`hardware.md` RAM table** contradicted itself (Model row "2 GB min" vs. RAM
    row "4 GB"). Collapsed to one row: 2 GB min, 4 GB recommended, 8 GB supported.
  - **Boot config** (`reference/respeaker-2mic-hat.md`) was missing `dtparam=spi=on`
    despite the APA102 LEDs running over SPI; added it to the `config.txt` block.
  - **"Single-threaded"** in `specs/state-machine.md` contradicted its own
    Concurrency table; reworded to "single-threaded control loop" with the
    transcription worker called out.
  - **`min_duration_seconds` semantics** were session-level in `configuration.md`
    but per-chunk in `recording.md`; reconciled to session-level so a short final
    chunk of a longer session is no longer silently dropped.
  - **Minor clarifications:** transcript **Duration** is derived from `session.wav`
    (not "all chunks", which are deleted); FR-18 notes the installer leaves
    `transcription.threads` at its default; the systemd contract notes
    `network.target` is ordering-only (per NFR-1) and that `systemctl restart` is
    the supported way to apply config changes.

## [1.0] — 2026-07-19

### Added
- **Initial RPi documentation set** — the target specification for the Raspberry
  Pi Earshot application (not yet built). Structured to mirror the ESP `esp32/`
  track: `requirements/`, `adr/`, `specs/`, `reference/`, `experiments/`.
- **Requirements** (`requirements/`): product scope, supported hardware (Pi 4B +
  ReSpeaker 2-Mic HAT), non-functional targets, connectivity, and the
  `open-technical-decisions.md` (TD-n) / `open-ux-questions.md` (UX-n) registries.
- **ADRs** (`adr/`): audio storage format, Python venv over Docker, hardware
  abstraction layer, systemd for service management, filesystem-as-state, and
  chunked recording — each with an *Implementation note* flagging intended
  implementation specifics.
- **Specs** (`specs/`): normative configuration schema, state machine + LED table,
  recording, storage/filesystem-state, transcription, and the installer + systemd
  service contract. `FR-n` identifiers give the implementation and `earshot-tui`
  stable requirement IDs to trace behavior to.
- **Reference** (`reference/respeaker-2mic-hat.md`): ReSpeaker 2-Mic HAT hardware
  facts and the expected WM8960 mixer/boot configuration.
- **Experiments** (`experiments/`): scaffolding (README + TEMPLATE) for future
  hardware validation.

### Decisions baked into the spec
- Capture is **mono** (the left mic), 16 kHz / 16-bit PCM.
- Transcription engine is **faster-whisper**.
- The `config.toml` schema is `[audio].alsa_pcm`,
  `[recording].shutdown_hold_seconds`, `[storage].data_dir` (not an
  `[encoding]`/`[shutdown]` schema).

### Audio format
- Audio is stored as a single **`session.wav`** (chunks concatenated at session
  end); `session.wav` is the transcribed artifact. See the
  [audio storage format ADR](adr/audio-storage-format.md).

### Notes
- Scope is **Pi 4B + ReSpeaker**.
