# Configure the application

## Purpose

Set runtime values without editing application source.

## Audience

Deployment operators and framework administrators.

## Prerequisites

Access to the Windows launcher or Docker environment and permission to restart the application.

## Settings

| Variable | Purpose | Default |
|---|---|---|
| `PORT` | HTTP port | `3399` |
| `EMU_APP_TITLE` | Display title | `EmuFramework` |
| `EMU_DB_PATH` | Business SQLite file | deployment default |
| `EMU_DESIGNER_DB_PATH` | Designer SQLite file | deployment default |
| `EMU_SECRET_KEY_PATH` | 32-byte integration encryption key file | `.emu-secret.key` beside `designer.db` |
| `EMU_SECURE_COOKIES` | Require HTTPS cookies | production dependent |
| `EMU_UPDATER_TOKEN` | App-to-sidecar secret | required for Docker update |
| `EMU_APP_CONTAINER` | Updater-only: name of the app container to restart | required for Docker update |
| `EMU_IMAGE_REPOSITORY` | Updater-only: app image repository to pull | `ghcr.io/emu479p01/emu-framework` |
| `EMU_UPDATE_STATE_PATH` | Updater-only: path to the update status file | `/data/update-status.json` |

Restart the app after changing environment values. Use HTTPS and `EMU_SECURE_COOKIES=true` on an internet-facing deployment.

## Manage report fonts

**Report Fonts** (`/system/fonts`, maintenance capability) lets an administrator install Google Fonts for offline use in the Report Designer and generated PDFs, without the report server needing internet access at render time.

1. Open **API settings** and save a Google Fonts Developer API key. The key is stored in the system database and is only ever shown masked afterward.
2. Choose **Sync catalog** to search and browse available families.
3. Choose **Install** on a family to download and cache it; installed, non-built-in fonts can be removed later.

Report Designer picks up installed fonts as a report-level default font or a per-element override. `Roboto` and `Noto Sans Thai` are built in and remain available without a configured API key. Generated PDFs automatically select `Noto Sans Thai` for Thai text.

## Configure SMTP

Only a user with `FW_SystemAdminRole` can manage the shared SMTP transport.

1. Open **Settings → SMTP Settings**.
2. Enter the SMTP host and port. Enable **Implicit TLS** for providers that require it, normally on port 465; port 587 normally starts without implicit TLS and upgrades the connection.
3. Enter the username and password when the server requires authentication.
4. Enter the sender address and optional sender name.
5. Select **Save**, then **Verify connection**.
6. Enter a recipient and select **Send test**.

Leaving the password blank during a later edit keeps the existing password. The password is encrypted in `designer.db`; it is never returned to the browser.

The encryption key is stored separately at `EMU_SECRET_KEY_PATH`, or in `.emu-secret.key` beside `designer.db` by default. Preserve the key in a secure host-level backup. A framework `.emubackup` does not contain it. If the key is lost, the stored SMTP password cannot be decrypted; restore the original key or save the password again.

The administrator API routes are:

```text
GET  /api/system/integrations/smtp
PUT  /api/system/integrations/smtp
POST /api/system/integrations/smtp/verify
POST /api/system/integrations/smtp/test
```

## Related topics

[Docker installation](docker-install.md) · [Troubleshooting](troubleshooting.md)
