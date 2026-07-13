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

Report Designer picks up installed fonts as a report-level default font or a per-element override; built-in fonts remain available even without a configured API key.

## Related topics

[Docker installation](docker-install.md) · [Troubleshooting](troubleshooting.md)
