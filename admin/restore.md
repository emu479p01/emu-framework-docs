# Restore databases

## Purpose

Restore the Full, Data, Designer, or Fonts components declared by a verified `.emubackup` package.

## Audience

Framework administrators responsible for recovery.

## Prerequisites

A validated `.emubackup`, Framework Administrator access, and a separate copy of the current databases and integration secret key.

## Procedure

1. Open **System Maintenance**, upload the package, and select **Preview restore**.
2. Review the manifest, selected components, checksums, SQLite integrity result, version, and warnings.
3. Keep a current Full backup and a separate copy of `.emu-secret.key` or `EMU_SECRET_KEY_PATH`.
4. Type `RESTORE` and confirm. Keep the page open while the app stops and reconnects.
5. Sign in again if Data was restored; user, security, and session records now match the backup.
6. Verify apps, recent records, Designer customizations, reports/fonts, and **Settings → SMTP Settings** for the components restored.
7. Verify the SMTP connection and send a test email. If the original key is unavailable, save the SMTP password again to encrypt it with the current key.

On supported Windows and Docker deployments the restore coordinator stages the validated files, stops the app, replaces only the selected components, restarts it, and performs a health check. If replacement or restart fails, it restores the pre-restore files and attempts to restart the previous state.

Never replace a live SQLite file manually. Do not mix components from different backup generations unless you have verified their compatibility. A `.emubackup` never contains the integration secret key; recover it separately.

## Related topics

[Backup](backup.md) · [Recovery](recovery.md)
