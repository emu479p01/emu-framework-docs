# Release notes

These notes describe the changes since v0.1.0.0 that affect users, administrators, and application developers.

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
- Designer and CLI object creation require an explicit App and Model. The CLI adds `pnpm emu add model <app> <name> --layer <layer>`.
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

[Install on Windows](admin/windows-install.md) · [Install with Docker](admin/docker-install.md) · [Configuration](admin/configuration.md) · [Functions and actions](developer/functions.md) · [Security](developer/security.md) · [Views and Charts](developer/views-and-charts.md)
