# Restore databases

## Purpose

Restore the components declared by a verified `.emubackup` package: Data, Designer, Fonts, Files, or Archive.

## Audience

Framework administrators responsible for recovery.

## Prerequisites

A validated `.emubackup` of 512 MB or less, Framework Administrator access, and a separate copy of the current databases and integration secret key.

## Procedure

1. Open **Settings → System Maintenance**, select **Upload & Restore**, and choose the package. The server validates it and opens a restore dialog.
2. Review the components the package contains, the warnings, and the version in the validation result. Validation checks the manifest, checksums, SQLite integrity, and that the package was not made by a newer framework version.
3. Keep a current Full backup and a separate copy of `.emu-secret.key` or `EMU_SECRET_KEY_PATH`.
4. Type `RESTORE` and select **Restore and Restart**. The preview expires after 10 minutes; upload the package again if it does. Keep the page open while the app stops and reconnects.
5. Sign in again if Data was restored; user, security, and session records now match the backup.
6. Verify apps, recent records, Designer customizations, reports and fonts, attachments, and **Settings → SMTP Settings** for the components restored.
7. Verify the SMTP connection and send a test email. If the original key is unavailable, save the SMTP password again to encrypt it with the current key.

On the supported Docker deployment, the updater sidecar stages the validated files, stops the app, takes a snapshot of the components to be replaced, replaces only those components, restarts the app, and performs a health check. If replacement or restart fails, it restores the snapshot and restarts the previous state. The restore job reports the same `phase`, `rollbackStatus`, and `recoveryRequired` fields as an update; if the rollback also fails, see [Recover from a failed update](recovery.md).

Restore Data and Files from the same moment. Attachment records live in the Data component and the files they point to live in the Files component, so restoring only one of them can leave attachments that cannot be opened. The same applies to Data and Archive.

Older packages remain restorable. Packages created before schema version 3 restore Data, Designer, and Fonts; packages from schema version 3 and 4 restore the components they declare.

Never replace a live SQLite file manually. Do not mix components from different backup generations unless you have verified their compatibility. A `.emubackup` never contains the integration secret key; recover it separately.

## Related topics

[Backup](backup.md) · [Recovery](recovery.md) · [Storage and archive](storage-and-archive.md)
