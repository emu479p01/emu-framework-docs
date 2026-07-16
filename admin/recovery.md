# Recover from a failed update

## Purpose

Return the framework to a known-good runtime while preserving evidence and database backups.

## Audience

Framework administrators and deployment operators.

## Prerequisites

Access to application logs, the previous framework version, and the latest verified backup.

## Procedure

1. Record the update job error and backup path from **System Maintenance**.
2. Check available disk space and application/updater logs.
3. Windows: inspect the timestamped framework backup and run `Update.cmd` after correcting the cause.
4. Docker: verify whether the sidecar restored the previous container, then follow the manual rollback procedure.
5. Start the previous framework version and verify data before retrying.
6. Restore the `.emubackup` only if data verification fails or release notes explicitly require database rollback.
7. Restore `.emu-secret.key` or the configured `EMU_SECRET_KEY_PATH` separately. After returning to a version that supports SMTP, verify the connection and send a test message.

Do not mix framework files from multiple releases. Preserve the failed state until logs and backups have been collected.

## Related topics

[Framework update](framework-update.md) · [Restore](restore.md)
