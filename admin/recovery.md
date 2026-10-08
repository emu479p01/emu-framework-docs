# Recover from a failed update

## Purpose

Return the Docker runtime to a known-good state while preserving evidence and verified backups.

## Audience

Framework administrators and Docker operators.

## Prerequisites

Access to the Compose project, app and updater logs, the previous image tag, the pre-update `.emubackup`, and the separately stored secret key.

## Read the job status first

Open **Settings → System Maintenance**, or read `/data/update-status.json` (updates) and `/data/restore-status.json` (restores). Three fields tell you what happened:

| Field | Meaning |
|---|---|
| `status` | `pending`, `running`, `restarting`, `succeeded`, or `failed` |
| `phase` | The last phase reached: `preparing`, `stopping`, `snapshotting`, `switching`, `restoring`, `verifying`, `completed`, `rolling_back`, `rolled_back`, or `recovery_required` |
| `rollbackStatus` / `recoveryRequired` | Whether the automatic rollback ran (`not_required`, `running`, `succeeded`, `failed`) and whether an administrator must finish recovery |

Choose the case that matches.

### Failed, no rollback needed (`status` is `failed`, `phase` is `completed`, `rollbackStatus` is `not_required`)

The updater stopped before it changed the app container or the data, for example because the image is not published or the app and updater mounts differ. The running version is unchanged. Fix the cause shown in `error`, then retry. See [Troubleshoot the system](troubleshooting.md).

### Failed and rolled back (`phase` is `rolled_back`, `rollbackStatus` is `succeeded`)

The new version did not become healthy. The updater restored the previous container and the pre-update data snapshot, so your data is as it was before the attempt. Read the `error`, check the app logs for the new version, and retry only after the cause is fixed.

### Recovery required (`recoveryRequired` is true, `phase` is `recovery_required`)

The automatic rollback could not finish. The recovery files stay in `/data/update-recovery-<job id>` (or `/data/restore-recovery-<job id>` for a restore). Do not delete them until the system is verified.

1. Preserve app and updater logs before recreating anything:

   ```sh
   docker compose logs --no-color app updater > emuframework-update.log
   ```

2. Check disk space, GHCR access, Docker socket access, updater token equality, container names, network membership, and the `/data`, `/files`, and `/archive` mounts.
3. Check whether the app container is running and healthy, and whether a leftover `...-rollback-...` container exists:

   ```sh
   docker ps -a --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
   ```

4. Set `EMU_VERSION` to the previous known-good tag and recreate the stack:

   ```sh
   docker compose pull
   docker compose up -d --force-recreate
   docker compose ps
   ```

5. Verify both SQLite databases and application data. If the databases or attachments are damaged, restore the pre-update `.emubackup` (Full) as described in [Restore databases](restore.md), and restore `.emu-secret.key` separately when needed.
6. Verify login, Apps, recent records, Designer customizations, attachments, SMTP, Scripts and Functions, and **System Maintenance** diagnostics.

### The updater was restarted during a job

When the updater starts, it looks for update and restore jobs left in `pending`, `running`, or `restarting`. It restarts the app if nothing had changed, rolls back from the snapshot if one exists, and otherwise marks the job `recovery_required`. Wait for the updater to become healthy (`docker compose ps`) and re-read the job status before taking any action.

## Rules that always apply

- Rolling back the image after a **successful** update does not undo database migrations. Restore the backup only when data verification fails or the release notes require it.
- Never run the old and new images against the same volumes at the same time.
- Never delete `-wal` or `-shm` files while a writer is active.
- Never use `docker compose down -v` during recovery; it deletes the named volumes.

## Related topics

[Framework update](framework-update.md) · [Restore](restore.md) · [Docker operations](docker-operations.md) · [Troubleshooting](troubleshooting.md)
