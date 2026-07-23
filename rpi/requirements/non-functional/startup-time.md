# Startup time

## Requirement

| SBC | Target: power-on → green-light ready |
|---|---|
| Pi 4B | 60 s |

"Ready" means the device is idle and able to accept a recording — the LED is solid green
and the button is live.

## Interaction with crash recovery

Startup also finalizes any orphaned sessions
([Resilience](resilience.md), [`specs/storage.md`](../../specs/storage.md#crash-recovery)).
Whether that work falls inside this 60 s budget is **not yet specified**: several
interrupted multi-hour sessions could take longer to encode than the target allows.

Either recovery precedes "ready" and is excluded from this target, or it runs behind a
device that is already green. This needs deciding before the target can be tested
meaningfully.
