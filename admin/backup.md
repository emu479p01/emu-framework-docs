# Back up databases

## Purpose

Create a verified package containing the recovery components you select.

## Audience

Framework administrators.

## Prerequisites

Framework Administrator access and enough storage outside the application host.

## Procedure

1. Open **System Maintenance** as a Framework Administrator.
2. Select **Full**, **Data**, **Designer**, or **Fonts**, then export the backup.
3. Store the `.emubackup` file outside the application host.
4. Use **Validate Restore File** to verify its manifest and checksums.

Component contents:

| Component | Contents |
| --- | --- |
| Full | Data, Designer, and Fonts |
| Data | `data.db`, including users, Roles, App Access, and sessions |
| Designer | `designer.db`, including runtime customizations and integration settings |
| Fonts | Uploaded/cached report font files |

Every `.emubackup` includes `manifest.json`, component declarations, and checksums. Use Full for disaster recovery; use a component backup only when the matching partial restore is intentional. Packages intentionally exclude `.emu-secret.key` or the file configured by `EMU_SECRET_KEY_PATH`.

If SMTP is configured, copy the integration secret key to a separate protected backup. The database contains the encrypted SMTP password, while the separate key is required to decrypt it. Store the key with access controls appropriate for a credential, but do not place it inside the `.emubackup` package.

Schedule additional host-level copies according to your recovery requirements.

## Related topics

[Restore](restore.md) · [Application data](app-data-management.md) · [Database storage](database-storage.md)
