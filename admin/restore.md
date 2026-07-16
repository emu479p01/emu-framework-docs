# Restore databases

## Purpose

Replace both databases from a verified `.emubackup` package.

## Audience

Framework administrators responsible for recovery.

## Prerequisites

A validated `.emubackup`, application stop/start access, and a copy of the current databases.

## Procedure

1. Validate the package in **System Maintenance**.
2. Stop the application and keep the current databases as an additional copy.
3. On Windows run `RestoreDatabase.cmd "C:\path\backup.emubackup"`.
4. For Docker, extract the validated database files into the stopped `emu-data` volume using an administrative container.
5. Restore the original integration secret key to `.emu-secret.key` beside `designer.db`, or to `EMU_SECRET_KEY_PATH`, before starting the app.
6. Start the app and verify login, apps, recent records, and **Settings → SMTP Settings**.
7. Select **Verify connection** and send a test email. If the original key is unavailable, save the SMTP password again to encrypt it with the new key.

Never replace a live SQLite file by copying over it. Restore both databases from the same package. Do not assume `.emubackup` contains the integration secret key; it must be recovered separately.

## Related topics

[Backup](backup.md) · [Recovery](recovery.md)
