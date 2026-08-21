# Recover from a failed update

## Purpose

Return the Docker runtime to a known-good state while preserving evidence and verified database backups.

## Audience

Framework administrators and Docker operators.

## Prerequisites

Access to the Compose project, app/updater logs, the previous image tag, the pre-update `.emubackup`, and the separately stored secret key.

## Procedure

1. Record the update job error, target version, and backup path from **System Maintenance** or `/data/update-status.json`.
2. Preserve app and updater logs before recreating a container:

   ```sh
   docker compose logs --no-color app updater > emuframework-update.log
   ```

3. Check disk space, GHCR access, Docker socket access, updater token equality, container name, network membership, and the `/data` mount.
4. Verify whether the updater already restored the previous container and whether it is healthy.
5. If needed, set `EMU_VERSION` to the previous known-good tag, then run:

   ```sh
   docker compose pull
   docker compose up -d --force-recreate
   docker compose ps
   ```

6. Verify both SQLite databases and application data before retrying the update.
7. Restore the `.emubackup` only when data verification fails or the release notes explicitly require database rollback. Restore `.emu-secret.key` separately when needed.
8. Verify login, Apps, recent records, Designer customizations, SMTP, Scripts/Functions, and **System Maintenance** diagnostics.

Rolling back the container image does not undo database migrations. Never run the old and new images against the same volume simultaneously, never delete `-wal` or `-shm` files while a writer is active, and never use `docker compose down -v` during recovery.

## Related topics

[Framework update](framework-update.md) · [Restore](restore.md) · [Docker operations](docker-operations.md)
