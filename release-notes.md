# Release notes

These notes describe the changes since v0.1.0.0 that affect users, administrators, and application developers. The current version is 1.4.0. The same notes are published with each [GitHub release](https://github.com/emu479p01/emu-framework/releases).

## About version numbers

Starting with v1.0.0, releases use three components, `Major.Minor.Patch`, and each release has a type:

- **FU (Framework Update)** adds functionality or makes structural changes. A breaking structural change increments `Major`; important backward-compatible functionality increments `Minor`.
- **PU (Proactive Update)** contains bug fixes, hotfixes, and security fixes, and increments `Patch`.

The framework updater accepts only stable `X.Y.Z` targets. Legacy four-component versions, such as v0.5.0.0 below, remain immutable history. The Compose file defaults to the current release tag instead of `latest`.

## v1.4.0

**FU - Framework Update.** Released 2026-09-28.

### Draft-based record creation

- Opening **New** on a form no longer inserts a row. The form shows default values and values set by `initValue`, including read-only document numbers, before you save. Saving is atomic and retry-safe, and a draft survives a server restart.
- Drafts are encrypted, belong to the user who opened them, and expire after 24 hours. An expired draft, another user's draft, or a draft created before a metadata change is rejected with `409`; reopen **New** to continue. The values you typed stay in the open page but are not transferred automatically.
- Every table exposes read-only audit aliases `sys_createdBy`, `sys_createdAt`, `sys_modifiedBy`, and `sys_modifiedAt` for lists, filters, sorting, and Views.

### Display and branding

- Datetimes are stored and returned in UTC and shown in the browser time zone. Datetime input without an offset is interpreted as UTC. Numbers display with grouping; IDs, references, and strings are not reformatted.
- `EMU_APP_TITLE` sets the product name in the browser title and on the login and setup pages.
- Deployment import previews start collapsed with add, change, remove, and high-risk counts, and show where moved or deleted artifacts came from.

### Functions with image input

- A Function can declare `imageInput` to receive JPEG, PNG, or WebP images attached to a record, chosen from files or taken with the camera. Uploads are signature-checked, permission-checked, and retried per file. WebP attachments can be previewed.

### Breaking change

Lifecycle hooks (`initValue`, `validateWrite`, `validateDelete`) and data event handlers must be synchronous. An `async` function or a returned Promise now raises `async lifecycle handlers are not supported`. Move awaited work into an async Function. System field names are reserved case-insensitively, and schema sync stops without changes if an existing column collides with an audit alias.

Before upgrading, review Scripts for async lifecycle handlers, create a Full backup, and preserve `.emu-secret.key`. Update the app and updater images together to `1.4.0`. Draft storage (`FW_RecordDraft`) is created automatically and no data migration is required. See [Troubleshoot the system](admin/troubleshooting.md) for the new failure messages.

## v1.3.0

**FU - Framework Update.** Released 2026-09-22.

### App-local navigation and Apps & Models

- Recent navigation is app-local: each App (and Settings) keeps its own list of up to ten entries per user, following the original menu structure and including record detail routes. Existing Favorites data is preserved.
- **Settings → Apps & Models** lists Apps and Models with layer, source, metadata revision, dependencies, deployment history, and license state. See [Manage Apps, Models, deployments, and licenses](admin/apps-models-licenses.md).

### Selected-model deployment and licenses

- The Designer builds deployment packages for selected Models: **Vendor update** (SYS, ISV, LOC layers) or **UAT → Prod** (all layers). Import preview shows additions, changes, removals, and preserved Models; ownership conflicts, missing dependencies, and incompatible customizations block deployment. Unselected Models and business data are kept.
- Applying a change set revalidates the candidate and restores runtime registrations and database changes when a synchronous apply or persistence step fails.
- Optional offline ISV licenses use Ed25519 files bound to a customer ID and installation ID, issued with the seller CLI. When a required license is missing, invalid, or expired, the owning App and its dependent Apps become read-only; a valid renewal takes effect without redeployment.

Before upgrading, back up the data and designer databases together and update the app and updater images together to `1.3.0`. License tables are created automatically in `designer.db`, and existing Models remain unlicensed. Selected-model packages use schema version 2 and require 1.3.0 at the destination; version-1 packages remain importable. Each environment keeps its own installation identity, so promote metadata with packages and never clone a designer database.

## v1.2.0

**FU - Framework Update.** Released 2026-09-19.

### Navigation and language

- **Recent** replaces Favorites in the UI. It always shows the ten most recently opened authorized menu items, with the newest first. The star buttons and the Favorites root are gone; the Favorites API and stored data remain.
- Framework screens ship in English and Thai. Choose the language from the user menu; routes, open records, and unsaved form state are kept.
- Apps can declare a **Default language**. Label translation falls back from the user's exact locale to its base language, the App's default language, and then the stored label.

### Designer and attachments

- The Designer adds a Translation editor and structured Data Entity and Data Entity Extension editors.
- Image attachments (PNG and JPEG) show thumbnails and open in a responsive viewer on desktop and mobile.
- System Maintenance separates **Storage Overview** from **Archive Policy**, with labeled usage rows, per-mount capacity, storage paths, and a confirmation before every manual archive run.

Before upgrading, create a Full backup and update the app and updater images together to `1.2.0`. No schema migration is required. After the restart, set each App's Default language in the Designer if it is not English.

## v1.1.0

**FU - Framework Update.** Released 2026-09-19.

### Attachments and storage

- Saved header and line records accept file, note, and URL attachments with SHA-256 integrity, streaming upload and download, and the permissions of the parent record. Limits are set with `EMU_ATTACHMENT_MAX_BYTES` and `EMU_ATTACHMENT_ALLOWED_TYPES`.
- Docker deployments gain the `emu-files:/files` and `emu-archive:/archive` volumes with `EMU_FILE_STORAGE_PATH` and `EMU_ARCHIVE_STORAGE_PATH`. Existing installations keep working with `/data/files` and `/data/archive`, and the administration page shows a warning.
- A Storage & Archive dashboard reports filesystem capacity and usage for databases, WAL files, backups, fonts, live files, the archive, and each App.
- Backup schema version 4 adds the **Live attachments** and **Archive catalog and payloads** components, and backups stream to the browser.

### Data Entities, archiving, and reports

- Header/Line Data Entities import and export documents as XLSX or a CSV ZIP package. Each document commits atomically and failed rows can be downloaded.
- Archive policies for archive-eligible Data Entities move old documents out of the live tables into immutable, checksummed payloads. Archived documents can be searched read-only and restored.
- The Report Designer adds A3, A4, A5, Letter, Legal, and custom paper sizes, orientation, four margins, display units, borders, and PNG/JPEG images from bundled assets or record attachments.
- Metadata labels can be translated per locale, and each user can choose a locale.

Before upgrading, create a Full backup and preserve `.emu-secret.key`. Add the two new volumes and environment variables, then update the app and updater images together to `1.1.0`. Before it stops the app, the updater verifies that both containers see the same file and archive mounts. Plan capacity for databases, backups, attachments, and the archive; see [Manage storage and archiving](admin/storage-and-archive.md). Archive data is kept indefinitely because automatic purge is not provided.

## v1.0.2

**PU - Proactive Update.** Released 2026-09-19.

### Durable updates and restores

- Updates and restores run through a phase-based maintenance journal. Status records add the optional `phase`, `rollbackStatus`, and `recoveryRequired` fields; the existing `status` field is unchanged.
- When health verification fails, the updater restores the previous container and the pre-operation data snapshot. Interrupted phases are compensated idempotently, and recovery artifacts are kept when automated rollback cannot finish.
- Both SQLite databases are checkpointed and closed on graceful shutdown.
- Update targets must be stable `X.Y.Z` versions. A local-image mode for unpublished test images fails closed.
- Unauthorized form and line actions are omitted from `/api/metadata`; direct action calls still return `403`.
- Report Designer canvas width honors asymmetric left and right margins.

Before upgrading, back up both databases, preserve `.emu-secret.key`, and update the app and updater images together to `1.0.2`. No migration is required. See [Update the framework](admin/framework-update.md) and [Recover from a failed update](admin/recovery.md).

## v1.0.1

**PU - Proactive Update.** Released 2026-09-03.

### Safer updates

- The updater exposes a health endpoint and a container health check.
- An image that is not published is rejected before the running app container is changed.
- If an update fails after the restart begins, the previous container is restored, and background update failures are recorded reliably.
- The updater detects interrupted update and restore jobs on restart, recovers the app container where possible, and marks the job failed with an actionable message.

### AI Proposal Inbox

- Adds status filters, refresh, expand and collapse controls, and responsive cards. Reviewed proposals can be removed from the Inbox without removing applied metadata or audit records.

Before upgrading, back up both databases, preserve `.emu-secret.key`, and update both images to `1.0.1`. No migration is required.

## v1.0.0

**FU - Framework Update.** Released 2026-09-03.

### First release under the new version model

- Establishes EmuFramework 1.0.0 as the baseline for the metadata-driven platform described in these docs: Web Designer, generated business UI on SQLite, layered customization, role-based security, REST integrations, reviewed AI proposals, and Docker deployment with backup, restore, and health diagnostics.
- This is the first release under the three-component `Major.Minor.Patch` version model. The legacy four-component tags up to `0.5.0.0` stay immutable and are not extended.

Before upgrading an existing installation, back up both SQLite databases and preserve `.emu-secret.key`. Migrations are idempotent.

## v0.5.0.0

### Scale and Web Designer

- A single Docker instance now supports 5,000+ mixed metadata Artifacts through a single-pass dependency pipeline, content-hash caching, affected-table schema synchronization, indexed Designer metadata, and incremental persistence.
- Artifact listing is paginated separately from the effective catalog, supports App/Model/kind filters and ETags, and keeps large Designer workspaces responsive.
- Form Extensions can add Line grids or override inherited line fields, aggregates, actions, labels, visibility, and order while persisting only the current Layer's `lineOverrides` delta.

### AI proposal workflow

- Scoped, versioned AI REST endpoints replace the removed MCP package. AI clients can inspect metadata, read schemas, validate ChangeSets, and submit proposals containing any supported Artifact, including Scripts and Functions.
- AI tokens are hashed, optionally expiring and revocable, restricted to selected Apps, and scoped to `inspect`, `validate`, and `propose`.
- AI cannot apply a ChangeSet or read business records. A customizer must review, revalidate, approve, or reject each proposal in the Designer Proposal Inbox; token and review activity is audited.

### Runtime and database operations

- Production deployment is Docker-only. The user CLI, MCP package, Windows host launchers/installers, and host update/restore scripts have been removed.
- SQLite remains the only database. Both `data.db` and `designer.db` use WAL, foreign keys, a 5-second busy timeout, automatic checkpoints, and integrity diagnostics.
- Documented v0.1.x metadata, Function, Script, backup, and synchronous `DataContext` behavior remains compatible. Direct access to private handles such as `kernel.db.prepare()` is not a compatibility guarantee.

Before upgrading, stop every older writer, create a Full backup, and preserve `.emu-secret.key` separately. After migration, verify login, important business records, Scripts/Functions, inherited Form Lines, the Proposal Inbox, and **Settings → System Maintenance** database diagnostics.

## v0.1.6.1

### Paginated report fixes

- Header and Footer space is reserved only on pages where the configured `displayOn` policy renders the band.
- Freeform Detail and Line rows remain atomic when moved across a page break; each main Detail is followed by its Line sources.
- Tablix pagination honors `headerHeight` and `rowHeight`, repeats headers after page breaks, and keeps Footer coordinates inside the physical page.

## v0.1.6.0

### Menu Extension editor

- The Designer shows inherited and current-Layer menu items in one effective tree while saving only the current Layer delta.
- Menu Extensions support visibility, target and order overrides, reset-to-inherited, and `parentId` anchors for added items.
- Stable IDs remain in metadata and backups but are hidden from the normal visual editor. Visibility is applied before privilege filtering and never grants authorization.

## v0.1.5.0

### Business tables and reports

- Desktop and mobile record lists use the same horizontally scrollable table with visible remote pagination.
- Menu Extensions can insert delta-only items into inherited submenus through stable-ID paths.
- Paginated reports add Tablix Detail and Line layouts, repeated headers, field formatting, native PDF pagination, and unified Header/Footer display policies.

## v0.1.4.0

### Layered customization and migration

- Designer now shows inherited layers as read-only and saves only the current Layer delta. Tables, Enums, Forms, Menus, Views, Charts, Privileges, Duties, Roles, Scripts, and Functions can be customized through Extensions.
- New `viewExtension`, `chartExtension`, and `functionExtension` kinds join the existing Extension kinds. Presentation overrides can change labels, visibility, order, icons, field editability, and chart measure presentation without copying a base artifact.
- Form groups, actions, Charts, lines, and menu items use stable element IDs so higher-layer overrides continue to target the intended element after lower layers change.
- The one-time `v0.1.4.0-layered-customization` migration normalizes legacy metadata, canonicalizes Extension names when safe, records audit copies, and is idempotent. Business records are not changed.

### Business UI and reports

- Enum and read-only fields must be optional. Read-only values remain writable from trusted Functions and Scripts, but generated Forms and REST writes cannot edit them.
- Form line create/update/delete actions now require confirmation. A reusable two-axis business grid adds sticky headers and row/column actions for dense business data.
- PDF rendering uses grapheme-safe, glyph-aware fallback for mixed Thai and Latin text.

Before upgrading, export a Full backup and retain `.emu-secret.key` separately. After upgrading, inspect migration diagnostics, open every inherited customization, and test read-only fields, line actions, Views, Charts, Functions, and mixed-language reports.

## v0.1.3.0

### App data management and web restore

- Framework administrators can export, replace, or permanently delete all business data owned by one App under **Settings → App Data Management**.
- `.emuappdata` packages preserve record IDs and validate the App name, framework compatibility, checksums, table schemas, and cross-App references. Replace is atomic and bypasses per-record business hooks.
- System Maintenance can export and restore Full, Data, Designer, or Fonts `.emubackup` packages. Restore preview validates checksums and SQLite integrity before confirmation.
- Docker restores stop, replace, restart, health-check, and roll back automatically. Restoring Data also restores users, security, and sessions from that backup.
- Desktop App navigation uses a one-branch accordion and remembers the last open branch for the browser session. Mobile navigation remains unchanged.

See [Manage application data](admin/app-data-management.md), [Back up databases](admin/backup.md), and [Restore databases](admin/restore.md).

## v0.1.2.0

### Mobile usability and portable packages

- Form controls use a mobile-safe 16px input size so iPhone Safari does not zoom automatically when a field receives focus; manual pinch-to-zoom remains available.
- App and Settings icons stay centered when the desktop sidebar is collapsed.
- App and Model metadata packages can be imported up to 20 MB, allowing larger framework exports to be imported again.

## v0.1.1.0

### Deny-by-default security

- Runtime access now requires both App Access (`canOpen`) and Role → Duty → Privilege permission. `FW_SystemAdminRole` is the only global bypass.
- `canCustomize` is independent. It opens only the granted business App in Designer and grants no App entry or business-data access. `FW_FrameworkUser` remains only as a legacy marker.
- User, Role assignment, App Access, session, migration, Designer-artifact, and View-token tables are blocked from generic data, import, and export endpoints.
- System Administrators manage users through **Settings → Users & Security**. Usernames are immutable, assignments are selected from validated Roles and Apps, and the last enabled System Administrator is protected.
- Every user can change their own password with current-password verification. Administrators can reset a password. Both flows require at least 12 characters and revoke old sessions; password and token hashes are omitted from responses.

See [Manage users and application access](admin/user-security.md) and the complete [permission matrix](developer/security.md).

### Explicit Apps and Models

- Every new App starts with `models: []`, including Apps named `erp`, `erp.credit`, or `web`. No App name creates a hidden or default Model.
- Add a named Model and choose its Layer before creating artifacts. App is the runtime access and navigation boundary; Model groups development metadata and is not a security boundary.
- Designer object creation requires an explicit App and Model.
- Framework metadata is shown to System Administrators in a separate **Framework — Read-only** area and cannot be edited, deleted, imported, exported, or extended.

Upgrade migrations materialize Models referenced by legacy artifacts and preserve old ERP/web metadata only for upgraded data. The one-time migration ledger prevents a later boot from granting access or recreating legacy Models again.

### Declarative Views and embedded Charts

- View artifacts support validated source aliases, inner/left equality joins, typed parameters, bound filters, grouping, sorting, and `count`, `sum`, `avg`, `min`, and `max` aggregates without raw SQL.
- Interactive View calls enforce App Access, View privilege, source-table read privileges, and row scopes.
- JSON is paged at 1,000 rows by default and capped at 10,000 per request. CSV exports default to a configurable cap of 100,000 rows.
- High-entropy service tokens are stored as hashes, scoped to named Views, optionally expiring, revocable, and accepted only in the Bearer header for View endpoints.
- Reusable bar, line, pie, donut, and KPI Chart artifacts can be embedded in Forms with record/literal parameter bindings. Apache ECharts is bundled locally and renders responsively.

See [Build Views and embedded Charts](developer/views-and-charts.md) and [Connect Power BI to the View API](admin/power-bi-view-api.md).

### Compatibility notes

- Existing `FW_FrameworkUser` assignments receive `canCustomize=true` only for business Apps that existed during upgrade; existing `canOpen` is preserved and new Apps require a new grant.
- Users without App Access are denied after upgrade unless they hold `FW_SystemAdminRole`.
- Generic `/api/data/FW_*` access to protected security tables now returns an authorization/not-found response by design.
- Back up and test the security matrix, legacy Model migration, password flows, Views, service tokens, and Charts before upgrading production.

## v0.1.0.2

### Secure administrator setup

New installations no longer use a shared default password. On first start, the server prints a one-time setup code. Open the setup page, enter that code, choose an administrator username and display name, and set a password of at least 12 characters.

The code expires after 15 minutes or ten failed attempts. Restart the server to generate a new code. A username must contain 3–60 letters, numbers, dots, underscores, or hyphens.

An upgraded installation that still has `admin` / `admin` is forced through the same setup page. In this legacy-reset case the username remains `admin`, all existing sessions for that account are removed, and the password must be replaced.

Administrator authority is now determined exclusively by membership in `FW_SystemAdminRole`. The username `admin` has no built-in privilege. Review role assignments after upgrading, especially for accounts that previously relied on their username.

### SMTP and integration services

Framework administrators can configure the shared mail transport under **Settings → SMTP Settings**. The page supports saving the connection, verifying it, and sending a test message.

The SMTP password is encrypted in `designer.db` with a separate 256-bit key. By default, the server creates `.emu-secret.key` beside `designer.db`; set `EMU_SECRET_KEY_PATH` to store it elsewhere. A `.emubackup` deliberately excludes this key, so preserve it through a separate secure backup. If the key is missing after a move or restore, copy the original key back or enter the SMTP password again.

Functions can use two reviewed integration services:

- `services.http.request(...)` for bounded HTTP or HTTPS requests.
- `services.email.send(...)` for mail through the configured SMTP transport.

Choose `executionMode: "async"` for a Function that uses `await`. Async Functions do not hold one automatic database transaction open while waiting for a network service. Use short explicit `ctx.tts()` blocks for database work and never await network I/O inside them.

### Upgrade checklist

1. Create and validate a full backup before updating.
2. Preserve `.emu-secret.key` separately if SMTP has been configured.
3. Start v0.1.0.2 and complete the setup page if prompted.
4. Confirm every administrator who needs unrestricted access has `FW_SystemAdminRole`.
5. Verify App Access separately from Role/Duty/Privilege permissions.
6. Verify SMTP, send a test email, and exercise every async Function.

## v0.1.0.1

### Mobile and responsive interface

The application shell, action picker, master-detail line grids, import workflow, Table Browser, Report Fonts page, Web Designer, and Report Designer now adapt to small screens. Tables that are difficult to use on a phone switch to record cards or provide horizontal scrolling. The Report Designer keeps a fixed-width design canvas inside a horizontally scrollable viewport.

### Thai report output

`Noto Sans Thai` is bundled as a built-in report font and does not require a Google Fonts API key. Generated PDFs automatically use it for Thai text, including when another font is selected for surrounding Latin text. It is also available in Report Designer for preview and explicit selection.

## Related topics

[Install with Docker](admin/docker-install.md) · [Configuration](admin/configuration.md) · [AI REST API](developer/ai-rest-api.md) · [Functions and actions](developer/functions.md) · [Security](developer/security.md) · [Views and Charts](developer/views-and-charts.md)
