# USB Offload

Retrieve recordings by inserting a FAT32-formatted USB stick into a USB-A port on
the Pi 4B (FR-11). LED behavior: [state-machine.md](state-machine.md).

## FR-11: USB stick offload

### Detection
- A background thread (`earshot-usb`) polls every **2 s** for a removable device.
- On insertion: the offload-pending flag is set and any error state cleared.
- On removal: pending and error state are cleared.

### Timing
- **Inserted while idle:** offload begins immediately.
- **Inserted during a recording session:** the stick is registered and offload is
  deferred. Recording continues normally (LED stays **red**). When the button
  ends the session and the final chunk finishes encoding, offload begins.
- **Inserted during transcription:** transcription is cancelled and offload runs.
- **Already inserted at boot:** offload triggers once the device reaches idle.

### Transfer
1. Wait up to ~10 s for the systemd mount to appear (the OS auto-mounts the
   stick). If it never mounts, skip offload and return to idle.
2. LED pulses **blue**.
3. Session directories are moved to the stick one at a time
   (`move_recordings_to_stick`): write to stick → delete from Pi.
4. Partial/crashed sessions (chunk WAVs not yet concatenated) are moved as-is.
5. On success: the stick is ejected (`eject_usb_device`), the LED **flashes blue
   once**, and the device returns to solid **green**. Safe to remove.

### Error handling
- If the stick fills mid-transfer (`ENOSPC`): the transfer stops, remaining
  recordings stay on the Pi, and the LED pulses **orange**. The error state
  clears when the stick is removed.
- Other `OSError`s are logged and surface the same orange error state.

### Stick requirements
- Filesystem: **FAT32**.
- No drivers or software on the receiving computer — standard USB mass storage.

> **Note.** Offload is **not** gated on transcription. A session offloads whether
> or not `transcript.md` exists; a `require_transcript_before_offload` option is
> deferred ([backlog B-T1](../requirements/backlog.md)).
