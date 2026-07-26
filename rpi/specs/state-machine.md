# State Machine

The application is a single-threaded control loop driven by the ReSpeaker button and
the web UI ([capabilities](../requirements/web-ui/README.md)); transcription is offloaded to a
cancellable worker thread (see [Concurrency](#concurrency)). On the ReSpeaker HAT the
LED is the **local** feedback channel; the web UI is the detailed one.

## LED states

| State | RGB | Colour | Pattern |
|---|---|---|---|
| Booting | `255,255,255` | White | Slow pulse |
| Ready / idle | `0,255,0` | Green | Solid |
| Recording | `255,0,0` | Red | Snap to solid, then slow pulse |
| Finalizing (encode) | `255,180,0` | Amber | Slow pulse |
| Processing (transcribe / diarize) | `255,179,0` | Amber | **Very** slow pulse (~1.5–2 s) |
| Disk threshold reached | `255,128,0` | Orange | Slow pulse |
| Recording too short (discarded) | `0,255,0` | Green | Double flash, then solid |
| Shutting down | `255,255,255` | White | Slow pulse → fade to off |

> Amber (`#FFB300`/`#FFB400`) is deliberately distinct from warning orange
> (`#FF8000`): more yellow, and the processing variant pulses slower.

### Pattern definitions
| Pattern | Description |
|---|---|
| Solid | Constant on |
| Slow pulse | Smooth fade in/out, ~1 s cycle |
| Very slow pulse | Smooth fade in/out, ~1.5–2 s cycle |
| Single/double flash | Sharp on/off, brief |
| Fade to off | Slow brightness decrease to off |

## FR-1: Idle
- On startup the LED pulses **white** while booting; on ready it goes solid **green**.
- If the disk threshold is already reached at startup, the LED pulses **orange**
  and the device waits for files to be removed (it does not accept recordings).
- The app polls the button and serves the web UI.
- Transcription (optionally diarized) is **initiated from the web UI**
  ([capabilities](../requirements/web-ui/README.md)) — it has no button gesture.
  Transcription runs locally by default; diarization is an option that requires a configured
  processing service. While a job is in flight the device is in **Processing**; at most one
  runs at a time ([processing.md](processing.md#processing-jobs)).

## FR-2: Start recording
- A button press while idle begins a session, provided the disk threshold is not
  reached (if it is, the press is ignored and the LED stays orange).
- The web UI can start a recording as well
  ([recording control](../requirements/web-ui/recording-control.md)); button and web are
  equivalent and
  act on the single active session.
- **Local transcription yields to recording.** If one is running when a recording starts
  (button or web), it is cancelled immediately, the session returns to the **front** of the
  queue, and recording begins without delay — it holds CPU on the capturing machine.
- **A job on a [processing service](../reference/processing-service.md) does not.** The work is on
  another machine, so it continues alongside the recording, neither degraded nor
  cancelled ([processing.md](processing.md#preemption)).
- The LED goes to **red** (snap solid, then slow pulse) for the session duration.
- Capture spec: **16 kHz, 16-bit PCM, mono** (left mic). Details:
  [recording.md](recording.md).
- Minimum duration (default 3 s) is enforced — a shorter recording is discarded
  and the LED **double-flashes green**.

## FR-3: Stop recording
- A second button press — or a stop from the web UI — ends the session
  (subject to minimum duration).
- The LED goes **amber** (finalizing) while the chunks are concatenated and encoded into
  a single `session.m4a`, then returns to solid **green**.
- If the encode fails, the error is logged and the chunk WAVs are retained; the
  LED still returns to green and recovery is retried on next boot.
- Button presses and web start/stop actions are ignored during post-recording
  processing — new recordings are blocked until the device is idle again.

## Creating a session by upload
The web UI can also create a session from an uploaded audio file
([upload an audio file](../requirements/web-ui/upload-audio.md)). The device decodes and
encodes it to `session.m4a` immediately, showing the same **Finalizing (encode)** amber
state while it runs, then returns to idle with the session **pending**. **Upload is refused
while a recording is active** — capture is sacred and the encode holds CPU on the Pi.

## FR-4: Safe shutdown
- Holding the button for `shutdown_hold_seconds` (default 3 s) **while idle**
  initiates safe shutdown.
- The LED goes to slow-pulsing **white**, then fades to off when it is safe to
  unplug.
- Shutdown is requested via `reboot(2)` `LINUX_REBOOT_CMD_POWER_OFF` (requires
  `CAP_SYS_BOOT`), falling back to `systemctl poweroff --no-wall`.
- Button holds during recording or processing are ignored.

## Concurrency
| Thread | Role |
|---|---|
| Main | State loop: idle ↔ record ↔ finalize ↔ process ↔ shutdown |
| `earshot-job-*` | Drains the job queue — spawning a subprocess for local transcription, or submitting to the processing service and polling it ([job execution](../adr/job-execution.md)) |

At most one job worker exists at a time ([processing.md](processing.md#processing-jobs)).
A **service** job coexists freely with **Recording**; a **local** one is cancelled by it.
