# Boot Configuration

v1 does not require user configuration for recording, labeling, listing, playback, deletion, or sync seams. However, the firmware should establish a small configuration-file scaffold now so future settings can be added without changing boot/storage structure.

## File location

The boot configuration file lives at the SD-card root:

```text
/earshot.cfg
```

## v1 behavior

- On boot, after SD mount, firmware should attempt to read `/earshot.cfg`.
- If the file is missing, firmware should create a default scaffold file.
- If the file is present, firmware should parse it permissively.
- Missing, unreadable, malformed, or unknown configuration must never block core
  recorder operation.
- v1 has no required config keys.

## File format

Plain UTF-8 / ASCII text, one `key=value` pair per line.

- Blank lines are ignored.
- Lines beginning with `#` are comments.
- Unknown keys are ignored and preserved where practical.
- Invalid values are ignored independently; one bad key must not invalidate the
  whole file.

## Default scaffold

If `/earshot.cfg` does not exist, create a minimal commented file such as:

```ini
# earshot configuration
# v1 has no required settings.
# Future settings may be added here as key=value lines.
```

Failure to create the scaffold should be logged where possible, but must not be a
fatal error.

See also `storage.md`.
