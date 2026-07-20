# State Machine

The application is a single-threaded control loop driven by the ReSpeaker button and
the web UI ([web-ui.md](../requirements/web-ui.md)); transcription is offloaded to a
cancellable worker thread (see [Concurrency](#concurrency)). On the ReSpeaker HAT the
LED is the **local** feedback channel; the web UI is the detailed one.

## LED states

| State | RGB | Colour | Pattern |
|---|---|---|---|
| Booting | `255,255,255` | White | Slow pulse |
| Ready / idle | `0,255,0` | Green | Solid |
| Recording | `255,0,0` | Red | Snap to solid, then slow pulse |
| Finalizing (concatenation) | `255,180,0` | Amber | Slow pulse |
| Transcribing | `255,179,0` | Amber | **Very** slow pulse (~1.5–2 s) |
| Disk threshold reached | `255,128,0` | Orange | Slow pulse |
| Recording too short (discarded) | `0,255,0` | Green | Double flash, then solid |
| Shutting down | `255,255,255` | White | Slow pulse → fade to off |

> Amber (`#FFB300`/`#FFB400`) is deliberately distinct from warning orange
> (`#FF8000`): more yellow, and the transcription variant pulses slower.

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
- Transcription and diarization are **initiated from the web UI**
  ([web-ui.md](../requirements/web-ui.md)). While a web-initiated transcription runs,
  the device is in **Transcribing** (see [transcription.md](transcription.md)).

## FR-2: Start recording
- A button press while idle begins a session, provided the disk threshold is not
  reached (if it is, the press is ignored and the LED stays orange).
- The web UI can start a recording as well (FR-23); button and web are equivalent and
  act on the single active session.
- If transcription is running when the button is pressed, it is cancelled
  immediately, the in-progress session returns to the **front** of the queue, and
  recording begins without delay.
- The LED goes to **red** (snap solid, then slow pulse) for the session duration.
- Capture spec: **16 kHz, 16-bit PCM, mono** (left mic). Details:
  [recording.md](recording.md).
- Minimum duration (default 3 s) is enforced — a shorter recording is discarded
  and the LED **double-flashes green**.

## FR-3: Stop recording
- A second button press — or a stop from the web UI (FR-23) — ends the session
  (subject to minimum duration).
- The LED goes **amber** (finalizing) while chunks are concatenated into a single
  `session.wav`, then returns to solid **green**.
- If concatenation fails, the error is logged and the chunk WAVs are retained; the
  LED still returns to green and recovery is retried on next boot.
- Button presses and web start/stop actions are ignored during post-recording
  processing — new recordings are blocked until the device is idle again.

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
| Main | State loop: idle ↔ record ↔ finalize ↔ transcribe ↔ shutdown |
| `earshot-transcribe-*` | Runs one session's transcription; cancellable via an event |
