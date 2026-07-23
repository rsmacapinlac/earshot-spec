# Start and stop recording

Start and stop a recording from the web UI, shared with the hardware button.

**This is parity, not an extension.** Record, stop, and status are the same capability on
two surfaces. The button and the web UI act on the single active session, and the LED and
state machine reflect the same state regardless of which control acted
([state-machine.md](../../specs/state-machine.md)).

- A web start is subject to the same conditions as a button press — notably the disk
  threshold, which blocks new recordings.
- Minimum duration is enforced identically: a session shorter than the minimum is
  discarded whichever control stopped it.

Contrast with transcription and diarization, which exist **only** on the web surface — the
button has no gesture to spare, and should not acquire one.
