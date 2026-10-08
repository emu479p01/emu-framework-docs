# Back up databases

## Purpose

Create a verified package containing the recovery components you select.

## Audience

Framework administrators.

## Prerequisites

Framework Administrator access and enough storage outside the application host. A package that includes attachments and the archive can be as large as those folders combined.

## Procedure

1. Open **Settings → System Maintenance** as a Framework Administrator.
2. In **Database backup**, choose a component under **Data to back up**: **Full**, **Data database**, **Designer database**, **Report fonts**, **Live attachments**, or **Archive catalog and payloads**.
3. Select **Download Backup**. The browser downloads `emuframework-<component>-<version>-<timestamp>.emubackup`.
4. Store the `.emubackup` file outside the application host.
5. Select **Upload & Restore** with the package to check its manifest, checksums, and SQLite integrity. You can close the dialog without restoring. See [Restore databases](restore.md).

The server streams the package to the browser as it builds it, so large backups do not need to fit in memory.

Component contents:

| Component | Contents |
| --- | --- |
| Full | Data, Designer, Fonts, Files, and Archive |
| Data | `data.db`, including users, Roles, App Access, sessions, attachment records, and new-record drafts |
| Designer | `designer.db`, including runtime customizations, integration settings, and ISV license data |
| Fonts | Uploaded and cached report font files |
| Files | Live attachment files from `EMU_FILE_STORAGE_PATH` |
| Archive | The archive catalog and archived payloads and attachments from `EMU_ARCHIVE_STORAGE_PATH` |

Every `.emubackup` includes `manifest.json`, component declarations, and checksums. The current package format is schema version 4. Use Full for disaster recovery; use a component backup only when the matching partial restore is intentional. Back up **Data** and **Files** together: a Data backup restores attachment records, and without the matching Files component the records can point to files that do not exist.

Packages intentionally exclude `.emu-secret.key` or the file configured by `EMU_SECRET_KEY_PATH`.

If SMTP is configured, copy the integration secret key to a separate protected backup. The database contains the encrypted SMTP password, while the separate key is required to decrypt it. Store the key with access controls appropriate for a credential, but do not place it inside the `.emubackup` package.

## Size limit

The web restore and the pre-update backup validate each package in memory and reject packages larger than 512 MB with `Backup exceeds the 512 MB safety limit`. If your attachments and archive make a Full package larger than that, take component backups (for example Data and Designer together, and Files and Archive separately) and add host-level copies of the volumes for disaster recovery. See [Manage storage and archiving](storage-and-archive.md).

Schedule additional host-level copies according to your recovery requirements.

## Related topics

[Restore](restore.md) · [Application data](app-data-management.md) · [Database storage](database-storage.md) · [Storage and archive](storage-and-archive.md)
