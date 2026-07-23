# Clock independence

## Requirement

**Identity, ordering, and labelling must never depend on the system clock being correct.**

A recording captured on a device that has never known the time must be fully usable:
identifiable, orderable, nameable, playable, and processable. A wrong clock may degrade
what is *displayed*; it must not affect what anything *is*.

## Why

The Raspberry Pi 4B has **no real-time clock** ([hardware.md](../hardware.md)). It keeps time only while powered, and relies on the network to learn it again at boot.

Capture, meanwhile, must work with no network at all ([no internet; graceful degradation](no-internet-graceful-degradation.md)). Those two facts collide directly: a device that cold-boots without a network has no time source — systemd restores a coarse saved timestamp — so anything stamped in that window is stamped from a clock known to be wrong.

The button makes this unavoidable rather than unlikely. A recording started from the hardware button involves no browser, no network, and no external time source of any kind. There is nothing better to consult, so the dependency has to be removed rather than supplied.

## What this constrains

| | Must not | Must instead |
|---|---|---|
| Session identity | Be derived from a timestamp | Be an allocated identifier that consults no clock |
| Queue order | Sort by capture time | Sort by identity, which is monotonic |
| Transcript header | Carry a recording date | Carry the session name, or the ID |
| Stored timestamps | Be read back for any decision | Be descriptive metadata only |

The last row is the general rule: **the device may record what time it thinks it is, but
nothing may act on it.** Nothing sorts, looks up, recovers, or establishes identity by
time.

## Where this is satisfied

- [`adr/session-identity.md`](../../adr/session-identity.md) — the decision that satisfies
  it, and the alternatives rejected for reintroducing a clock dependency.
- [`specs/storage.md`](../../specs/storage.md#session-identity) — allocation, and
  `created_at` as descriptive metadata.
- [`specs/processing.md`](../../specs/processing.md#time-independence) — queue ordering and
  the transcript header.

## Not in scope

Making the clock **correct**. NTP, an RTC module, and user-set time are all out of scope —
this requirement exists precisely so that none of them is needed. If an RTC were added
later, nothing here would change; the displayed dates would simply be more trustworthy.
