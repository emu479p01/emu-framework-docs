# Understand database storage

## Purpose

Plan capacity for the SQLite business and Designer databases and for the volumes that hold attachments and archived data.

## Audience

Deployment operators and database administrators.

## Prerequisites

Host access to the Docker installation and a verified backup before storage changes.

## What is stored where

SQLite files grow automatically as records and indexes are added. There is no database-size setting to increase. Available space is controlled by the disk or VM that stores Docker volumes.

| Location | Volume | Contents |
|---|---|---|
| `data.db` | `emu-data` (`/data`) | Business and framework records, users and sessions, attachment records (`FW_Attachment`, `FW_Blob`), archive policies and data job history, and encrypted new-record drafts (`FW_RecordDraft`) |
| `designer.db` | `emu-data` (`/data`) | Designer metadata and state, AI tokens, proposals, audit records, and ISV license data: installation identity, trusted vendor keys, installed licenses, and license audit |
| `backups/`, `fonts/`, status files | `emu-data` (`/data`) | Pre-update and server-side backups, installed report fonts, and update and restore status |
| Attachment files | `emu-files` (`/files`) | Live attachment content under opaque storage keys |
| Archive | `emu-archive` (`/archive`) | `catalog.db` (a SQLite catalog), archived document payloads, and archived attachment files |

Both databases run in WAL mode with foreign keys enabled, a 5-second busy timeout, and automatic checkpoints every 1,000 WAL pages. **Settings → System Maintenance** reports these values and runs integrity diagnostics. The app checkpoints and closes both databases when it receives a graceful stop (`SIGTERM`), so use `docker compose stop` rather than killing the container.

New-record drafts are small encrypted rows in `data.db`. A draft lasts 24 hours, its content is cleared once the record is saved, and expired drafts are deleted the next time any user opens a new draft. They need no capacity planning.

SQLite supports one writer at a time. Do not scale EmuFramework by running multiple writer containers against the same volumes.

The integration encryption key is not a SQLite file. It is stored at `EMU_SECRET_KEY_PATH`, or as `.emu-secret.key` beside `designer.db`. Keep it on persistent storage and back it up separately from `.emubackup` exports.

If `EMU_FILE_STORAGE_PATH` or `EMU_ARCHIVE_STORAGE_PATH` is not set, attachments and the archive fall back to `/data/files` and `/data/archive` on the data volume, and the Storage Overview shows a warning. See [Install with Docker](docker-install.md).

## Plan capacity

Size the disk for the databases and their WAL files, backups (a Full backup holds a compressed copy of everything), live attachments, archive payloads, and headroom for the snapshot the updater takes before an update or restore, which copies the databases, fonts, attachment files, and archive. A practical SSD baseline is:

| Deployment | Disk |
|---|---|
| Light use, no meaningful attachment growth | 100 GB |
| Typical server (4 vCPU, 8 GB RAM) | 200 GB |
| File-heavy workloads | 500 GB or more |

Review **Storage Overview** monthly. Archiving moves old documents out of the live tables but keeps them on disk; archived data is retained until you delete the volume content, because automatic purge is not provided. See [Manage storage and archiving](storage-and-archive.md).

## Check Docker usage

1. Run `docker system df` for host-wide usage.
2. Run `docker volume inspect emuframework-data emuframework-files emuframework-archive` to locate the default Compose volumes, or inspect the names set by `EMU_VOLUME_NAME`, `EMU_FILES_VOLUME_NAME`, and `EMU_ARCHIVE_VOLUME_NAME`.
3. Monitor free space on the Docker host or Docker Desktop virtual disk, and check the per-mount **Total** and **Free** values in Storage Overview.
4. Create a verified backup before changing disk or VM settings.
5. Increase the host filesystem or Docker Desktop disk image, then confirm free space.

Do not delete SQLite `-wal` or `-shm` files while the app is running. Do not run `docker compose down -v` unless permanent data deletion is intended.

## Related topics

[Backup](backup.md) · [Docker operations](docker-operations.md) · [Storage and archive](storage-and-archive.md)
