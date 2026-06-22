# State Machine

The canonical behavioural spec for earshot: the core states, their
transitions, the condition/interrupt screens, and what each button does in each.
This was previously captured only in the design chat and split across two
prototypes that had drifted; this document is the single source of truth for
behavior.

## Buttons

Two physical buttons, active-low with internal pull-ups (see `../reference/hardware-pinout.md`):

| Name | GPIO | Role |
| ---- | ---- | ---- |
| BOOT | 0    | cycles / advances / cancels (left slot in hints) |
| PWR  | 18   | confirms / activates (right slot in hints)       |

Hint position carries the button — left hint = BOOT, right hint = PWR — so
on-screen hints never print the button names, only the action and a modifier.

Press detection (from the device spec):

| Event  | Definition |
| ------ | ---------- |
| short  | release before the long threshold, not part of a double |
| long   | held past **600 ms** |
| double | two presses within a **200 ms** window |

The reference firmware uses a **350 ms** short-press debounce. Single/long/double
detection is delegated to the **OneButton** library, configured to these
thresholds — see `../adr/0004-button-input-handling.md`.

### On-screen hint glyphs

Each screen draws a two-cell button bar at the bottom: the **left** cell is BOOT,
the **right** cell is PWR. A cell shows the action label plus a glyph that encodes
the required gesture, so the button name is never printed:

| Glyph | Gesture |
| ----- | ------- |
| one dot `•` | single short press |
| two dots `••` | double press |
| bar `▬` | press & hold (long) |

A cell with no action for the current state draws a `-` placeholder. Long-press is
hinted in only two places per the transition table — `RECORDINGS_LIST` hold-left
to exit and hold-right to delete; the empty list / NO SD `EXIT` holds use the bar
glyph. Any global escape-hatch gesture remains unhinted pending **UX-1**.

## States

Seven logical states form the navigable flow below. Separately, a set of
**condition screens** (low battery, no SD, sleep, …) are raised by hardware
events rather than button navigation — see "Condition & interrupt screens".

`LIST` has a distinct **empty** presentation, so there are eight core screens.

| State          | Activity level    | Purpose |
| -------------- | ----------------- | ------- |
| IDLE           | light sleep target | Standby home: battery, free storage, note count. |
| RECORDING      | fully active      | Capturing audio; live elapsed timer. |
| LABEL_PROMPT   | fully active      | Post-save: offer to add a spoken label to the note just saved. |
| LABEL_CAPTURE  | fully active      | Capture/replace a note's spoken `label.wav`; same model as RECORDING. |
| RECORDINGS_LIST| fully active      | Browse stored notes, one selected at a time. |
| PLAYBACK       | fully active      | Play the selected note; elapsed/total + progress bar. |
| DELETE_CONFIRM | fully active      | Guard a destructive delete. |

`LABEL_PROMPT` and `LABEL_CAPTURE` implement the optional spoken-label feature
(see `storage.md` → "Labels and relabeling"). They are reached two ways: after a
recording saves (`RECORDING → LABEL_PROMPT`), and from `PLAYBACK` to relabel an
existing note (`PLAYBACK → LABEL_CAPTURE`).

## Transition table

| From | Catalyst | To / effect |
| ---- | -------------- | ----------- |
| **IDLE** | PWR double | RECORDING (start capture, elapsed = 0) |
| **IDLE** | BOOT double | RECORDINGS_LIST (listIndex = 0) |
| **IDLE** | (120 s idle) | enter sleep (see Power, below) |
| **RECORDING** | PWR short | **save** note → LABEL_PROMPT |
| **RECORDING** | BOOT short | **cancel** (discard take) → IDLE |
| **LABEL_PROMPT** | PWR short | `LABEL` → LABEL_CAPTURE (capture for the just-saved note) |
| **LABEL_PROMPT** | BOOT short | `SKIP` → IDLE (note is already saved; a label can be added later) |
| **LABEL_CAPTURE** | PWR short | `SAVE` → commit `label.wav` → return to caller |
| **LABEL_CAPTURE** | BOOT short | `CANCEL` → keep any prior label → return to caller |
| **RECORDINGS_LIST** (non-empty) | BOOT short | advance selection (wraps) |
| **RECORDINGS_LIST** (non-empty) | BOOT long | exit → IDLE |
| **RECORDINGS_LIST** (non-empty) | PWR short | PLAYBACK of selected (elapsed = 0) |
| **RECORDINGS_LIST** (non-empty) | PWR long | DELETE_CONFIRM |
| **RECORDINGS_LIST** (empty) | BOOT long | exit → IDLE |
| **RECORDINGS_LIST** (empty) | PWR short | exit → IDLE |
| **DELETE_CONFIRM** | PWR short | delete selected → RECORDINGS_LIST |
| **DELETE_CONFIRM** | BOOT short | cancel → RECORDINGS_LIST |
| **PLAYBACK** | BOOT short | `RELABEL` → stop playback → LABEL_CAPTURE (relabel this note) |
| **PLAYBACK** | PWR short | stop → RECORDINGS_LIST |
| **PLAYBACK** | playback reaches end | auto-finish → RECORDINGS_LIST |

### LABEL_CAPTURE caller / return target

`LABEL_CAPTURE` remembers how it was entered and returns there on both `SAVE`
and `CANCEL`:

- entered from **LABEL_PROMPT** (post-save) → return to **IDLE**;
- entered from **PLAYBACK** (`RELABEL`) → playback is stopped on entry; return to
  **RECORDINGS_LIST** with the same note still selected.


## Empty-list behaviour

- Entering RECORDINGS_LIST with zero notes shows the empty screen. `BOOT long → IDLE`
  is the displayed hint; `PWR short → IDLE` is also accepted as a harmless OK/exit path.
- Deleting the **last** remaining note returns to RECORDINGS_LIST in its empty
  presentation (not to IDLE). Deleting with notes remaining keeps the list and
  clamps the selection to a valid index.

## Condition & interrupt screens

These screens are raised by **hardware/system conditions**, not by navigating the
flow above. They are interrupts: an advisory one overlays and returns you where
you were; a blocking one prevents the action and drops to IDLE. Layouts are in
the design.
CHARGING is currently a reserved/renderable condition only: the firmware has no
charger-present detector wired, so it is not auto-raised in normal operation.

| Screen | Raised when | Kind | Action → result |
| ------ | ----------- | ---- | --------------- |
| LOW BATTERY | battery state enters LOW | advisory (non-blocking) | PWR dismiss → return to prior screen; recording continues |
| CRITICAL BATTERY | battery state enters CRITICAL; new recording is blocked, or active recording was gracefully stopped/saved | blocking | PWR `OK` → IDLE |
| CHARGING | reserved for charger-present detection; current firmware has no hardware signal/API wired | advisory when raised by test/future detector | PWR dismiss → return to prior screen |
| NO SD CARD | card absent / unreadable | blocking | BOOT `exit (hold)` → IDLE |
| STORAGE FULL | new recording cannot be created or started because storage is unavailable/full | blocking | PWR `OK` → IDLE |
| SLEEP | 120 s inactivity | transient | any button will wake to IDLE |

Notes:

- **Advisory screens don't abort the current activity** — dismissing LOW BATTERY
  during RECORDING returns to the live recording, not IDLE.
- CRITICAL BATTERY is blocking. If it happens during recording, firmware should
  first attempt graceful stop/save, then raise the blocking condition.
- LOW BATTERY / CHARGING currently require a keypress to dismiss. Whether they
  should **auto-return** after a moment instead is **UX-7** in
  `../requirements/open-ux-questions.md`.
- These interrupts are independent of the core transition table and the UX-1
  global-rule question.
- **No time/clock condition screen in v1.** Recordings are identified by ID and
  carry no timestamp (`storage.md`), so there is no RTC dependency and no
  "time not set" / "clock not set" advisory. The PCF85063 RTC remains a hardware
  reference only (`../reference/rtc-pcf85063.md`), not a v1 UI condition.

## Power interaction

earshot **boots directly into IDLE** — there is no splash/boot screen in v1. IDLE
is the only state that sleeps: after **120 s** of
inactivity the device enters light-sleep, drawing the SLEEP screen first. Any
button wakes it back to IDLE. Activity in any active state resets the inactivity
timer. The exact sleep depth (light vs deep) is provisional pending battery
testing.

See also `power-sleep.md`.
