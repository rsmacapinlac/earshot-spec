# Show device status

Show the live state the on-device LED reflects — booting, ready, recording, finalizing,
processing, disk threshold — continuously and on every view.

**Status is the third parity item**, alongside record and stop. The web UI shows what the
LED is showing, so:

- A user can never stop a recording from a page that never told them one was running.
- The **disk-threshold block stops being LED-only**. On a headless device an orange pulse
  is the sole indication that recordings are blocked, which is a poor way to learn it.

During a recording the LED shows **Recording** even if a service job is running alongside;
capture is the more important local signal, and the web UI names what else is happening
([state-machine.md](../../specs/state-machine.md#led-states)).

Live recording elapsed time belongs here too — the state alone does not say how long.
