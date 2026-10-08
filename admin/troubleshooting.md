# Troubleshoot the system

## Audience

Framework administrators and deployment operators.

## Prerequisites

Access to application status, logs, host storage, and Docker commands when applicable.

## App does not start

Check ports, free disk space, container state, and logs:

```sh
docker compose ps
docker compose logs --tail=200 app updater
```

In production the app stops at start-up if `EMU_DEPLOYMENT_MODE` is not `docker` or if `EMU_DB_PATH` or `EMU_DESIGNER_DB_PATH` is not under `/data/`. The log names the cause.

`docker compose ps` also shows each container's health. The app is healthy when `/api/health` answers; the updater when `/health` answers on port 3400. See [Operate Docker](docker-operations.md).

## Schema sync stops with "reserved audit aliases collide with stored columns"

Every table exposes the virtual fields `sys_createdBy`, `sys_createdAt`, `sys_modifiedBy`, and `sys_modifiedAt`. If an existing table has a physical column with one of those names (compared without regard to case), schema sync stops before it makes any change and names the table and columns. Ask the App developer to rename or remove the colliding field, then restart. See [Record lifecycle](../developer/record-lifecycle.md).

## Update check fails

Confirm access to GitHub Releases and GHCR. A proxy, DNS rule, API rate limit, missing updater token, or private package visibility can block the request. The error `Latest release tag must use X.Y.Z` means the newest GitHub release does not use a stable three-component tag, so the web update will not run.

## Update fails or is rejected

| Message or symptom | Cause and action |
|---|---|
| `Container image ... is not published. No container was changed.` | The target tag is not in the registry. Wait for publication or pick a published version. The running app is untouched. |
| `Persistent mount '/files' is missing or differs between app and updater; application was not stopped` | The updater and the app do not mount the same source at `EMU_FILE_STORAGE_PATH` or `EMU_ARCHIVE_STORAGE_PATH` (or at `/archive`). Mount the same volumes in both services, recreate both, and retry. See [Update the framework](framework-update.md). |
| `Cannot verify updater mounts; application was not stopped` | The updater could not inspect its own container. Recreate the updater container with Compose. |
| `Persistent .emu-secret.key or EMU_SECRET_KEY_PATH is required before update` | The secret key file does not exist at `EMU_SECRET_KEY_PATH` (default `/data/.emu-secret.key`). Restore the key from your separate copy, make sure the path is on a persistent volume, and retry. |
| `Invalid stable version; expected X.Y.Z` | The target is not a stable three-component version. |
| `Backup exceeds the 512 MB safety limit` | The pre-update backup, which includes attachments and the archive, is too large for the web update. Take external backups of the volumes and use the manual fallback in [Update the framework](framework-update.md). |
| `A framework update is already running` | Another job is `pending`, `running`, or `restarting`. Wait for it to finish. |
| Phase `rolled_back` | The update failed and was rolled back; your data is unchanged. Read the `error` and the app logs. |
| `recoveryRequired` true | Automatic rollback failed. Follow [Recover from a failed update](recovery.md). |

## Database errors

Stop writes, preserve the database and WAL files, and create a filesystem copy before attempting recovery. Never edit a production SQLite file directly.

Open **Settings → System Maintenance** and inspect `journal_mode`, `foreign_keys`, `busy_timeout`, WAL checkpoint settings, and the integrity result for both databases. Stop all old containers if a lock persists; never run two writer containers against the same volume.

## Storage warnings and attachment errors

- **Shared fallback storage warning in Storage Overview:** `EMU_FILE_STORAGE_PATH` or `EMU_ARCHIVE_STORAGE_PATH` is not set, so attachments or the archive use `/data/files` or `/data/archive`. Add the volumes and variables ([Install with Docker](docker-install.md)).
- **Filesystem shows "Unknown":** the server could not read capacity for that mount. Check that the path exists and is mounted.
- **Upload fails with `413` or `Attachment exceeds the configured size limit`:** the file is larger than `EMU_ATTACHMENT_MAX_BYTES` (25 MB by default). A reverse proxy limit can cause the same symptom.
- **Upload fails with `415` or `Attachment type ... is not allowed`:** the extension and MIME type are not both permitted. Check `EMU_ATTACHMENT_ALLOWED_TYPES`.
- **Archive batch reports a failed document:** open **Job history** under Archive Policy for the error. A document with attachments fails when the policy does not include attachments, and a document already in the archive is skipped as `Document is already archived`.

See [Manage storage and archiving](storage-and-archive.md).

## An App is read-only (license)

If users see `App '<name>' is read-only: <app>/<model>: <status>`, a required ISV license is missing, invalid, not yet valid, or expired. Open **Settings → Apps & Models**, check the model's license state and the license audit, and import a valid license. Reading, exports, backups, and license administration keep working. See [Manage Apps, Models, deployments, and licenses](apps-models-licenses.md).

## New record cannot be saved (409)

Opening **New** creates a short-lived draft. A save is rejected with `409` when:

- `Draft expired or unavailable. Open a new draft; your entered values have not been saved.` The draft is older than 24 hours, belongs to another user, or no longer exists.
- `Metadata changed. Open a new draft before saving; keep your entered values.` Someone applied a Designer change or deployment while the form was open.

Tell the user to reopen **New** and re-enter the values; the values typed in the open page are kept but are not transferred automatically. Repeated `409` errors right after a deployment are normal for forms that were already open.

## Script error: "async lifecycle handlers are not supported"

Since 1.4.0 the hooks `initValue`, `validateWrite`, `validateDelete`, and data event handlers must be synchronous. An `async` function, or a handler that returns a Promise, raises this error and the write fails. Ask the developer to remove `async`/`await` from the hook and move awaited or external work (HTTP, email) into an async Function. See [Hooks and data events](../developer/hooks-events.md).

## Related topics

[Recovery](recovery.md) · [Database storage](database-storage.md) · [Storage and archive](storage-and-archive.md)
