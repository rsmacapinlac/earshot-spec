# Delete a session

Deleting a session - removs its directory and everything in it — audio, transcript, and
labels — and freeing the disk immediately.

- **Confirmation is required.** This is the only destructive action in the UI and it is
  not recoverable.
- Freed space is reflected in the disk figure, and clears the threshold block if usage
  drops back below it
  ([storage.md](../../specs/storage.md#disk-space-management)).

Delete is **not** a processing job, so it stays available while one is running. A job that
completes after its session was deleted discards its result rather than recreating the
directory.
