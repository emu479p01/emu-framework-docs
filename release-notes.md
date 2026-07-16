# Release notes: v0.1.0.1 and v0.1.0.2

These notes describe the changes since v0.1.0.0 that affect users, administrators, and application developers.

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

[Install on Windows](admin/windows-install.md) · [Install with Docker](admin/docker-install.md) · [Configuration](admin/configuration.md) · [Functions and actions](developer/functions.md) · [Security](developer/security.md)
