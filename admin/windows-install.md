# Install on Windows

## Purpose

Run a release without installing Node.js globally.

## Audience

Windows operators and framework administrators.

## Prerequisites

A writable Windows folder and permission to run the release launcher.

## Procedure

1. Download and extract the latest release to a writable folder.
2. Run `RunApp.cmd`; allow the first run to download the pinned toolchain.
3. Open the displayed URL. On first run, the app redirects to **Administrator setup**.
4. Copy the one-time setup code from the **Emu-Server** window, then enter it on the setup page.
5. Choose an administrator username and display name. The username must contain 3–60 letters, numbers, dots, underscores, or hyphens.
6. Set a password of at least 12 characters and complete setup.
7. Create and validate a backup.

The setup code expires after 15 minutes or ten failed attempts. Restart the server to generate a new code. An upgrade that detects the legacy `admin` / `admin` credentials requires a password reset and keeps the username `admin`.

Use `status.cmd` to inspect the app and `StopApp.cmd` to stop it.

## Common errors

- A proxy or firewall may block the first toolchain download.
- Port 3399 or 5199 may already be in use.
- If the setup code expired, stop and restart the app, then use the newly printed code.

## Related topics

[Configuration](configuration.md) · [Framework update](framework-update.md)
