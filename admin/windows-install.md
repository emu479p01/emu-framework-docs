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
3. Open the displayed URL and sign in with `admin` / `admin`.
4. Change the default password and create a backup.

Use `status.cmd` to inspect the app and `StopApp.cmd` to stop it.

## Common errors

- A proxy or firewall may block the first toolchain download.
- Port 3399 or 5199 may already be in use.

## Related topics

[Configuration](configuration.md) · [Framework update](framework-update.md)
