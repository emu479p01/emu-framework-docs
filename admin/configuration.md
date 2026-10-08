# Configure the application

## Purpose

Set runtime values without editing application source.

## Audience

Deployment operators and framework administrators.

## Prerequisites

Access to the Docker deployment and permission to recreate the application containers.

## Settings

Set these on the container shown in each section. In the official `docker-compose.yml`, the app already receives the values marked "Compose".

### App container

| Variable | Purpose | Default |
|---|---|---|
| `PORT` | HTTP port (host port in Compose) | `3399` |
| `EMU_APP_TITLE` | Product name shown in the browser title and on the login and setup pages | `EmuFramework` |
| `EMU_DEPLOYMENT_MODE` | Must be `docker`; a production app refuses to start otherwise, and web update reports `unsupported` without it | `docker` (Compose) |
| `EMU_DB_PATH` | Business SQLite file; in production it must be under `/data/` | `/data/data.db` (Compose) |
| `EMU_DESIGNER_DB_PATH` | Designer SQLite file; in production it must be under `/data/` | `/data/designer.db` (Compose) |
| `EMU_SECRET_KEY_PATH` | 32-byte integration encryption key file | `.emu-secret.key` beside `designer.db` |
| `EMU_SECURE_COOKIES` | Require HTTPS cookies | on when `NODE_ENV=production` |
| `EMU_FILE_STORAGE_PATH` | Directory for live attachment files | `/files` (Compose); `/data/files` with a warning if unset |
| `EMU_ARCHIVE_STORAGE_PATH` | Directory for the archive catalog and archived payloads | `/archive` (Compose); `/data/archive` with a warning if unset |
| `EMU_ATTACHMENT_MAX_BYTES` | Maximum size of one uploaded attachment, in bytes | `26214400` (25 MB) |
| `EMU_ATTACHMENT_ALLOWED_TYPES` | Comma-separated list of allowed MIME types | PDF, DOCX, XLSX, PPTX, CSV, plain text, PNG, JPEG, WebP |
| `EMU_BACKUP_DIR` | Where pre-update and other server-side backups are written | `/data/backups` |
| `EMU_FONT_DIR` | Cache for installed report fonts | `/data/fonts` |
| `EMU_DATA_JOB_PATH` | Staging area for Data Entity imports | `data-jobs` beside `data.db` |
| `EMU_RESTORE_STAGE_DIR` | Staging area for restore previews | `restore-jobs` inside `EMU_BACKUP_DIR` |
| `EMU_VIEW_CSV_MAX_ROWS` | Maximum rows in one View CSV export | `100000` |
| `EMU_UPDATER_TOKEN` | App-to-sidecar secret | required for Docker update |
| `EMU_UPDATER_URL` | Internal updater URL used by the app | `http://updater:3400` (Compose) |
| `EMU_UPDATE_STATE_PATH` | Update status file read by the app | `/data/update-status.json` |
| `EMU_RESTORE_STATE_PATH` | Restore job status file shared with the updater | `/data/restore-status.json` |

### Updater container

| Variable | Purpose | Default |
|---|---|---|
| `EMU_UPDATER_TOKEN` | Shared secret; at least 24 characters and identical to the app's | required |
| `EMU_APP_CONTAINER` | Name of the app container to replace | `emuframework-app` |
| `EMU_IMAGE_REPOSITORY` | App image repository to pull | `ghcr.io/emu479p01/emu-framework` |
| `EMU_UPDATE_STATE_PATH` | Path to the update status file | `/data/update-status.json` |
| `EMU_RESTORE_STATE_PATH` | Path to the restore status file | `/data/restore-status.json` |
| `EMU_DATA_PATH` | Root of the data volume inside the updater | `/data` |
| `EMU_FILE_STORAGE_PATH` | Must point to the same volume as the app's value | `/data/files` if unset (`/files` in Compose) |
| `EMU_ARCHIVE_STORAGE_PATH` | Must point to the same volume as the app's value | `/data/archive` if unset (`/archive` in Compose) |
| `EMU_UPDATER_PULL_POLICY` | `always` pulls the target image from the registry; `never` skips the pull and uses an image that already exists on the Docker host | `always` |
| `EMU_UPDATER_LOCAL_IMAGE` | With `EMU_UPDATER_PULL_POLICY=never`, the local image name or ID to deploy instead of `<repository>:<version>` | unset |

### Compose-only variables

These are read by Docker Compose from `.env`, not passed to the containers.

| Variable | Purpose | Default |
|---|---|---|
| `EMU_VERSION` | Image tag for both the app and the updater | the release shipped with the Compose file (`1.4.0`) |
| `EMU_VOLUME_NAME` | Name of the data volume | `emuframework-data` |
| `EMU_FILES_VOLUME_NAME` | Name of the attachment volume | `emuframework-files` |
| `EMU_ARCHIVE_VOLUME_NAME` | Name of the archive volume | `emuframework-archive` |
| `EMU_NETWORK_NAME` | Name of the Docker network | `emuframework-network` |

Recreate the app container after changing environment values. Restarting an existing container does not apply new variables. Use HTTPS and `EMU_SECURE_COOKIES=true` on an internet-facing deployment. `EMU_UPDATER_TOKEN` must contain at least 24 characters and must match in the app and updater containers.

View JSON endpoints always cap one page at 10,000 rows. `EMU_VIEW_CSV_MAX_ROWS` controls the separate CSV export cap used by Power BI and other integrations; choose a limit that fits available memory and request time.

### Attachment limits

`EMU_ATTACHMENT_MAX_BYTES` must be a positive number; any other value falls back to 25 MB. An upload that exceeds the limit is rejected with `413`. `EMU_ATTACHMENT_ALLOWED_TYPES` replaces the built-in MIME list when set (for example `application/pdf,image/png,image/jpeg`); an empty value keeps the built-in list. A file must also have an extension that matches its MIME type, and a rejected type returns `415`. Do not set the limit above what the reverse proxy and the host's free space can handle. See [Manage storage and archiving](storage-and-archive.md).

### Local image mode for the updater

By default the updater pulls `ghcr.io/emu479p01/emu-framework:<version>` and rejects an update when that image is not published. For an acceptance test with an image that exists only on the Docker host, set `EMU_UPDATER_PULL_POLICY=never` and, optionally, `EMU_UPDATER_LOCAL_IMAGE` to the image name. The updater then never contacts the registry and fails the update if the image is missing locally. Do not use this mode for normal production updates. The update target must still be a stable `X.Y.Z` version. See [Update the framework](framework-update.md).

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

[Power BI View API](power-bi-view-api.md) · [Docker installation](docker-install.md) · [Storage and archive](storage-and-archive.md) · [Troubleshooting](troubleshooting.md)
