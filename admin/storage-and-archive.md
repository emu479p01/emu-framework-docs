# Manage storage and archiving

## Purpose

Monitor storage, set attachment limits, move old business documents into the archive, and plan capacity.

## Audience

Framework administrators and Docker operators. Everything on this page requires `FW_SystemAdminRole`.

## Prerequisites

- The `emu-files` and `emu-archive` volumes and the `EMU_FILE_STORAGE_PATH` and `EMU_ARCHIVE_STORAGE_PATH` variables, as described in [Install with Docker](docker-install.md).
- For archiving, at least one Data Entity marked **Archive eligible** with a **Business date field**. A developer or customizer creates it in the Designer; see [Exchange documents with Data Entities](../developer/data-entities.md).

## Storage Overview

Open **Settings → System Maintenance** and find the **Storage Overview** card. It lists:

| Row | What it measures |
|---|---|
| Main database and business data | Size of `data.db` |
| Transaction log of the main database | Size of the `data.db` WAL file |
| Metadata saved through the Designer | Size of `designer.db` |
| Transaction log of the Designer | Size of the `designer.db` WAL file |
| Backup files | Size of the backup folder (`/data/backups` by default) |
| Fonts for reports | Installed report fonts |
| Attachments in use | Live attachment files |
| Archived data and attachments | The archive folder |

Expand **Storage paths · Filesystem** to see, for the data, attachment, and archive mounts, the **Total** and **Free** capacity and the configured path. A mount whose capacity cannot be read shows **Unknown** instead of zero. Expand **Database usage per App** to see how much of `data.db` each App uses; each row says whether the figure is **measured from the database** or **estimated**. These figures are approximate and are not backup sizes.

A warning at the top of the card means the app is using shared fallback storage (`/data/files` or `/data/archive`) because the dedicated paths are not configured. Add the volumes to avoid mixing attachment and archive growth with the databases.

## Attachments

Users add file, note, and URL attachments to saved header and line records (see [Work with attachments](../user/attachments.md)). Files are stored under `EMU_FILE_STORAGE_PATH` with opaque storage keys and a SHA-256 checksum; their records live in `data.db`. Uploads and downloads stream, so large files do not load into memory.

| Setting | Default | Effect |
|---|---|---|
| `EMU_ATTACHMENT_MAX_BYTES` | 25 MB | Larger uploads are rejected with `413`. |
| `EMU_ATTACHMENT_ALLOWED_TYPES` | PDF, DOCX, XLSX, PPTX, CSV, plain text, PNG, JPEG, WebP | Comma-separated MIME types. A file's extension must also match its type; other uploads are rejected with `415`. |

Change them in `.env` or the container environment and recreate the app container. See [Configure the application](configuration.md). Only PNG, JPEG, and WebP attachments can be previewed in the browser; other files download. Image uploads from Functions must be JPEG, PNG, or WebP, are verified against the file signature, and share the same size limit.

Attachments are included in the **Live attachments** backup component (and in Full backups). Back up Data and Files together; see [Back up databases](backup.md).

## Archive Policy

Archiving moves old, complete documents (a header and its lines, optionally with their attachments) out of the live tables into immutable, checksummed files in the archive volume. Live reports and lists no longer include them. Archived documents can be searched read-only and restored. Archived data is kept indefinitely; there is no automatic purge.

### Set a policy

1. Open **Settings → System Maintenance** and find the **Archive Policy** card. If it shows that no archive-eligible Data Entity exists, create one in the Designer first.
2. Choose the **Data Entity** (the document type to archive).
3. Set the **Business date field**. This date or datetime field on the root table decides a document's age.
4. Set **Age days**: documents whose business date is on or before today minus this many days are eligible.
5. Set **Batch size** (1 to 10,000, default 100): the most documents one run archives.
6. Choose the **Schedule** (**Daily** or **Weekly**), the **Weekday** for weekly runs, and the **Timezone** (an IANA name such as `Asia/Bangkok`, the default). The schedule decides what calendar day counts as "today".
7. Select **Include attachments** if archived documents should keep their attachments. If it is off, a document that has attachments is not archived and is reported as failed.
8. Select **Enabled** to turn on automatic runs, then **Save policy**. Enabling shows a confirmation because a run can start at the next scheduler check.

The scheduler checks once an hour. A policy is due when the schedule matches the current day in its timezone and it has not already run that day, and each scheduled run archives one batch. Unsaved edits are flagged, and **Preview** and **Archive batch** stay disabled until you save.

### Preview and run a batch

1. Select **Preview**. It shows the number of eligible documents, the oldest business date, the cutoff, and the batch size.
2. Select **Archive batch** and confirm. The dialog shows the entity, cutoff, and batch size. Moving documents out of the live tables cannot be undone from this page; use restore to bring them back.

A manual run needs an enabled policy. The result reports how many documents were archived and how many failed. For each document the app writes the archive payload and catalog entry first, then deletes the live rows in one transaction and removes the live attachment files afterward, so a failure never leaves a document in neither place.

### Search, view, and restore

**Archived documents (read-only)** lists the most recent archived documents with their entity, business key, and business date. Select **View** to open the stored payload, or **Restore** to put the document back.

Restore verifies the payload checksum, then re-imports the header and lines through the Data Entity and copies attachments back to live storage. It is skipped when a live document with the same business key already exists. Restored documents are marked as restored in the catalog. Archives made before a Data Entity Extension was added still restore; a new mandatory field without a default makes the restore fail and keeps the archive unchanged.

Automation can search the catalog with `GET /api/system/archive/documents?entity=<name>&q=<text>`, where `q` matches the business key.

### Job history

**Job history** lists recent archive and import jobs with their time, type, entity, status, and any error. Scheduled runs appear as the user `archive-scheduler`.

## Capacity planning

The disk must hold the databases and WAL files, backups, live attachments, the archive, and headroom for the snapshot the updater takes before updates and restores. A practical baseline for a 4-vCPU, 8 GB server is 200 GB of SSD. Use 100 GB only for light deployments without material attachment growth, and 500 GB or more for file-heavy workloads.

- Check **Storage Overview** regularly and expand the disk before free space on any mount is low.
- Archiving frees space in the live tables and attachment folder but adds the same data to the archive volume, so it does not reduce total disk use. Use a larger disk, or a separate disk for `emu-archive`, for long retention.
- A Full backup includes live attachments and the archive and is validated in memory. Packages over 512 MB cannot be validated or restored on the web; see [Back up databases](backup.md#size-limit).
- Updating the framework needs free space for a copy of the databases, attachments, and archive.

## Common errors

- **"Document is already archived":** the business key already exists in the archive catalog; the live copy is left in place.
- **"Document has attachments but policy does not include them":** enable **Include attachments** or leave the document in the live tables.
- **"An enabled archive policy is required":** save the policy with **Enabled** selected before running a batch.
- **"Business key already exists" on restore:** a live document with that key is present; rename or remove it first.

## Related topics

[Back up databases](backup.md) · [Database storage](database-storage.md) · [Configure the application](configuration.md) · [Troubleshooting](troubleshooting.md)
