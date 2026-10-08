# Operate Docker

## Audience

Docker operators and framework administrators.

## Prerequisites

Docker Compose access to the EmuFramework installation.

## Common commands

```sh
docker compose up -d
docker compose stop
docker compose start
docker compose ps
docker compose logs --tail=200 app
docker compose logs --tail=200 updater
docker compose pull
```

Use `docker compose stop` (or `docker compose down`) rather than killing the containers. On a graceful stop the app checkpoints and closes both SQLite databases. `docker compose down` removes the containers and network; the named volumes (data, files, and archive) remain. Never add `-v` unless deleting all application data, attachments, and the archive is intentional.

## Verify the updater

The updater is intentionally not exposed on a host port. It is reached only by the app over the Compose network:

```text
http://updater:3400
```

Both images define a container health check, so `docker compose ps` shows `healthy` for each service when it is ready. The app is checked at `/api/health` and the updater at `/health` (port 3400, inside the container). The updater's `/health` returns `{"ok":true,"busy":false}`; `busy` is true while an update or restore is running.

Check both services and their logs:

```sh
docker compose ps
docker compose logs --tail=100 app
docker compose logs --tail=100 updater
```

Confirm that the app, rather than only the updater, has the required settings:

```sh
docker compose exec app printenv EMU_DEPLOYMENT_MODE
docker compose exec app printenv EMU_UPDATER_URL
```

Expected values are `docker` and `http://updater:3400`. Confirm the storage paths match the volumes the updater mounts:

```sh
docker compose exec app printenv EMU_FILE_STORAGE_PATH EMU_ARCHIVE_STORAGE_PATH
docker compose exec updater printenv EMU_FILE_STORAGE_PATH EMU_ARCHIVE_STORAGE_PATH
```

Recreate the app after changing `.env`:

```sh
docker compose up -d --force-recreate app
```

## Docker Desktop checks

When containers were created manually in Docker Desktop, verify that both containers share a user-defined network and that the app has a `/data` volume. The updater must have access to `/var/run/docker.sock`; this grants host-level Docker control and should be treated as a high-privilege component. See [Docker installation](docker-install.md) for the exact connectivity test command for manually created containers.

## Update status states

A successful update moves through this status sequence, tracked in the file set by `EMU_UPDATE_STATE_PATH` (default `/data/update-status.json`):

```text
pending -> running -> restarting -> succeeded
```

The same file records a `phase` that advances through `preparing`, `stopping`, `snapshotting`, `switching`, `verifying`, and `completed`. A failed update ends in `rolled_back` (with `rollbackStatus` `succeeded`) or `recovery_required` (with `recoveryRequired` true). Restores use the file set by `EMU_RESTORE_STATE_PATH` (default `/data/restore-status.json`) and add a `restoring` phase. See [Update the framework](framework-update.md) and [Recover from a failed update](recovery.md).

## Diagnose `fetch failed` during update checks

Common causes, roughly in order of likelihood:

- The app and updater containers are not on the same Docker network.
- The updater container is missing the network alias `updater`.
- The app's `EMU_UPDATER_URL` is not `http://updater:3400`.
- `EMU_UPDATER_TOKEN` differs between the app and the updater.
- The updater is not mounting `/var/run/docker.sock`.
- The updater is not mounting `emu-data` at `/data`.
- `EMU_APP_CONTAINER` does not match the app container's actual name (the default is `emuframework-app`).

If the update starts but fails immediately with `Persistent mount ... differs between app and updater`, the updater and the app do not share the same `/files` or `/archive` volume. Compare them with `docker inspect` on both containers and recreate with Compose.

## Manual rollback

A failed web update rolls back automatically, including the data snapshot. Use this procedure only to go back after an update that succeeded.

1. Keep the verified pre-update backup.
2. Set `EMU_VERSION` in `.env` to the previous known-good stable `X.Y.Z` version.
3. Run `docker compose pull && docker compose up -d`.
4. Restore databases only when a documented migration requires it; switching the image back does not alter database files.

## Related topics

[Docker installation](docker-install.md) · [Database storage](database-storage.md) · [Framework update](framework-update.md) · [Recovery](recovery.md)
