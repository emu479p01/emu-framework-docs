# Understand database storage

## Purpose

Plan capacity for the SQLite business and Designer databases.

## Audience

Deployment operators and database administrators.

## Prerequisites

Host access to Docker or the Windows installation and a verified backup before storage changes.

SQLite files grow automatically as records and indexes are added. There is no database-size setting to increase. Available space is controlled by the Windows disk or by the disk/VM that stores Docker volumes.

The integration encryption key is not a SQLite file. It is stored at `EMU_SECRET_KEY_PATH`, or as `.emu-secret.key` beside `designer.db`. Keep it on persistent storage and back it up separately from `.emubackup` exports.

## Check Docker usage

1. Run `docker system df` for host-wide usage.
2. Run `docker volume inspect emuframework_emu-data` to locate the volume.
3. Monitor free space on the Docker host or Docker Desktop virtual disk.
4. Create a verified backup before changing disk or VM settings.
5. Increase the host filesystem or Docker Desktop disk image, then confirm free space.

Do not delete SQLite `-wal` or `-shm` files while the app is running. Do not run `docker compose down -v` unless permanent data deletion is intended.

## Related topics

[Backup](backup.md) · [Docker operations](docker-operations.md)
