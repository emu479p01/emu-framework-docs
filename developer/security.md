# Understand security

EmuFramework uses layered authorization. A user receives one or more **Roles**; roles contain **Duties** and duties contain **Privileges**. A role may also reference privileges directly.

## Audience

Developers, application administrators, and security reviewers.

## Prerequisites

An application with defined metadata and an understanding of its users, roles, duties, and privileges.

## Roles and privileges

- **Role** is the assignment boundary for a user, such as `FW_FrameworkUser` or `FW_SystemAdminRole`.
- **Duty** groups related privileges so the same permission set can be reused by several roles.
- **Privilege** grants access to tables and forms, and may also grant named functions and reports.
- **App access** is separate from role membership. An `FW_AppAccess` entry controls whether a user may open an app and whether they may customize it.

Table permissions are operation-specific: read, create, update, and delete. Form permission controls whether a page can be opened; it does not replace the table permission required by the page.

## Functions and reports

Named server functions and reports are security artifacts, not merely navigation targets. Add each function and report to the appropriate privilege, then assign that privilege through a duty/role. The server must check the function/report privilege before executing it or generating output. A function or report that is absent from a user's privilege set must return an authorization error even when the caller invokes its URL directly.

## Menu and page visibility

The metadata response is filtered for the authenticated user. A menu item is visible only when its form, function, or report target is accessible. Empty sub-menus are removed, and an app with no visible menu items is omitted. This filtering improves usability but is not a security boundary: direct page navigation and API requests must still be authorized server-side.

Buttons for operations the current user cannot perform should remain visible as disabled/dim where the UI needs to communicate the unavailable operation. The disabled state is only a client-side affordance; the API must enforce the same permission independently.

## Framework users

`FW_FrameworkUser` is a self-service account role. It may read only its own `FW_User` record and may update only its password. It must not read or modify another user's record, `FW_UserRole`, or `FW_AppAccess` data, and must not use those records to manage other users.

`FW_SystemAdminRole` is the framework superuser role. It has full access to every app, page, table operation, named function, and report, including newly installed application metadata. The seeded administrator is equivalent for bootstrap/maintenance purposes; replace its default credentials immediately.

## Server-side boundary

Every read and write path must evaluate the authenticated session and authorization policy, including generic data APIs, import/export, named functions, reports, Designer endpoints, and maintenance endpoints. Never rely on hidden routes, filtered menus, disabled buttons, or client-provided role information.

Use secure cookies behind HTTPS, replace seeded credentials, keep updater tokens outside source control, and give Docker socket access only to the dedicated updater.

## Migration and compatibility risks

- Existing users whose permissions depended on broad `FW_FrameworkUser` access may lose access to role/app-access screens; grant `FW_SystemAdminRole` or an explicit administrative role where appropriate.
- Privileges created before function/report permissions were introduced need an explicit review and migration. Existing table/form permissions do not automatically imply function/report access.
- Custom menus may disappear when all of their children become unauthorized. This is expected; verify menu targets after importing metadata.
- Clients that assume every metadata response contains every table, action, or report must handle filtered collections and disabled operations.
- Test upgrades against a copy of the database, including legacy role assignments and existing `FW_AppAccess` rows, before production rollout.

## Verification checklist

Test at minimum: self-only framework-user reads, denied access to another user and that user's role/app-access rows, password-only updates, full system-admin access, hidden empty menus, disabled unauthorized buttons, and direct API denial for functions and reports. Run unit, server integration, and UI tests for both allowed and denied paths.

## Related topics

[Configuration](../admin/configuration.md) · [Testing](testing.md)
