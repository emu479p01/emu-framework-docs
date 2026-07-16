# Back up databases

## Purpose

Create one verified package containing business data and Designer metadata.

## Audience

Framework administrators.

## Prerequisites

Framework Administrator access and enough storage outside the application host.

## Procedure

1. Open **System Maintenance** as a Framework Administrator.
2. Select **Export Full Backup**.
3. Store the `.emubackup` file outside the application host.
4. Use **Validate Restore File** to verify its manifest and checksums.

The package contains `data.db`, `designer.db`, and `manifest.json`. It intentionally does not contain `.emu-secret.key` or the file configured by `EMU_SECRET_KEY_PATH`.

If SMTP is configured, copy the integration secret key to a separate protected backup. The database contains the encrypted SMTP password, while the separate key is required to decrypt it. Store the key with access controls appropriate for a credential, but do not place it inside the `.emubackup` package.

Schedule additional host-level copies according to your recovery requirements.

## Related topics

[Restore](restore.md) · [Database storage](database-storage.md)
