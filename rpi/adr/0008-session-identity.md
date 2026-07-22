# 0008 — Session Identity: Allocated ID, Not Timestamp

**Status:** Accepted

## Context

Every session artifact hangs off the session directory name: the FIFO transcription
queue orders by it, crash recovery scans it, the transcript references it, and the web UI
displays it. The original scheme named directories by capture time
(`<YYYYMMDDTHHMMSS>`), which makes identity depend on the system clock at the instant a
session starts.

The Raspberry Pi 4B has **no RTC** ([hardware.md](../requirements/hardware.md)), and
NFR-1 requires recording to work fully offline. A device that cold-boots without a network
has no time source — systemd restores a coarse saved timestamp — so a session started in
that window is named from a clock known to be wrong. The consequences are not cosmetic:
the FIFO queue mis-orders, and the wrong date propagates into the transcript.

The two ways a session can start differ in what is available to them:

| Initiation | Time source available |
|---|---|
| Hardware button (FR-2) | none — purely local |
| Web UI (FR-23) | the initiating browser's clock |

So no better time source can fix the button path, which is the primary one for a desk
appliance. The dependency has to be removed rather than supplied.

## Decision

Session identity is an **allocated sequence ID**, `rec-NNNNNN`, chosen without consulting
the clock. Allocation is `max + 1` over the existing `rec-*` directories, with an optional
`.next_id` file as a hint that is never authoritative. Contract:
[storage.md](../specs/storage.md#session-identity).

Capture time demotes to metadata: `status.json` keeps `recorded_at` as descriptive
information, re-derivable from the session directory's creation time if it is lost.
Nothing sorts, looks up, or recovers by time.

This matches the ESP32 track, which reached the same conclusion for the same reason —
stable ID as identity, optional human label on top.

## Consequences

- **Queue order is provably correct.** FIFO by ID is true capture order unconditionally,
  where FIFO by timestamp was wrong exactly when the clock was.
- **A clockless session is a first-class session** — identifiable, nameable, orderable,
  playable, transcribable. Only the displayed date degrades, and nothing reads it back.
- **Directory names lose at-a-glance chronology.** Acceptable: the UI leads with the
  session name (FR-29) and IDs still sort into creation order, which timestamps only
  approximated.
- **Crash recovery loses the in-memory start time**, so a recovered session falls back to
  the directory's creation time. Previously the start time survived in the directory
  name; it is now approximate for recovered sessions only.
- **`earshot-tui` compatibility is affected** if it assumes timestamp-named directories —
  the same class of coupling as the `"encoded"` status literal and the transcript header
  format.
- Only one session is ever active (FR-23), so allocation cannot race and needs no locking.

## Alternatives

- **Client-supplied time on web-initiated recordings** — the browser's clock is reliable
  and could stamp the session. Rejected as an identity mechanism: it covers only one of
  two initiation paths, and a button press may happen before any browser has ever
  connected. Retained as a possible way to *improve `recorded_at`* on an isolated LAN,
  which is a metadata question, not an identity one.
- **Provisional name, renamed once the clock syncs** — rejected: renaming a directory
  breaks the stable-identity invariant that crash recovery, the queue, and any held
  reference depend on.
- **Block recording until the clock is synced** — rejected: directly violates NFR-1, and
  makes the offline case fail rather than degrade.
- **Keep timestamps and mitigate downstream** (names, unverified-date badges) — rejected
  as the status quo that leaves queue ordering wrong. Mitigation makes a bad identity
  survivable; it does not make it correct.
