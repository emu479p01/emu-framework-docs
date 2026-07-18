# Understand security

EmuFramework v0.1.1.0 is deny-by-default. A normal user must pass two independent gates: **App Access** for the owning App and **Role → Duty → Privilege** for the requested object and operation. A role may also reference privileges directly.

## Audience

Developers, application administrators, and security reviewers.

## Prerequisites

An application with defined metadata and an understanding of its users, roles, duties, and privileges.

## Roles and privileges

- **Role** is the assignment boundary for a user, such as `FW_FrameworkUser` or `FW_SystemAdminRole`.
- **Duty** groups related privileges so the same permission set can be reused by several roles.
- **Privilege** grants table operations and named Forms, Functions, Reports, and Views. A Chart inherits access from its View.
- **App access** is separate from role membership. An `FW_AppAccess` entry controls whether a user may open an app and whether they may customize it.

Table permissions are operation-specific: read, create, update, and delete. Form permission controls whether a page can be opened; it does not replace the table permission required by the page.

## Permission matrix

Business-object authorization is `App canOpen AND Role/Duty/Privilege`. `canCustomize` is evaluated separately for Designer. `FW_SystemAdminRole` is the only global bypass.

| Assignment | App/menu visible | Business API | Designer |
| --- | ---: | ---: | ---: |
| No Role, no App Access | No | Denied | No |
| Role only | No | Denied | No |
| `canOpen` only | No usable object | Denied | No |
| `canCustomize` only | No | Denied | Granted App only |
| `canOpen` + matching Role/Privilege | Yes | Allowed operations only | No |
| `canOpen` + matching Role/Privilege + `canCustomize` | Yes | Allowed operations only | Granted App only |
| `FW_SystemAdminRole` | All | All business APIs | All business Apps; Framework metadata read-only |

The matrix applies to Forms, table CRUD, Functions, Reports, Views, imports, exports, and direct API calls. App is the runtime access and navigation boundary. Model is a metadata/layer grouping and never grants security access.

## Designer permission

`canCustomize=true` exposes Designer for only the named business App. It does not imply `canOpen`, table access, or any other runtime permission. Designer menus are derived from server capabilities instead of membership in `FW_FrameworkUser`.

System metadata is visible only to a System Administrator under **Framework — Read-only**. All Designer create, update, delete, package import/export, and Extension paths reject changes to System Apps and Models.

## Functions and reports

Named server functions and reports are security artifacts, not merely navigation targets. Add each function and report to the appropriate privilege, then assign that privilege through a duty/role. The server must check the function/report privilege before executing it or generating output. A function or report that is absent from a user's privilege set must return an authorization error even when the caller invokes its URL directly.

## Menu and page visibility

The metadata response is filtered for the authenticated user. A menu item is visible only when its form, function, or report target is accessible. Empty sub-menus are removed, and an app with no visible menu items is omitted. This filtering improves usability but is not a security boundary: direct page navigation and API requests must still be authorized server-side.

Buttons for operations the current user cannot perform should remain visible as disabled/dim where the UI needs to communicate the unavailable operation. The disabled state is only a client-side affordance; the API must enforce the same permission independently.

## Framework users and administrators

`FW_FrameworkUser` is a legacy marker in v0.1.1.0 and grants no global Designer or business-data bypass. During upgrade, each existing user with this Role receives `canCustomize=true` only for business Apps that already existed. Existing `canOpen` values are preserved, and Apps created after upgrade require an explicit new grant.

`FW_SystemAdminRole` is the single framework superuser Role. It has full access to every business App, page, table operation, named Function, Report, and View, including newly installed application metadata. Administrator authority is role-based: the username `admin` has no special access unless that user holds `FW_SystemAdminRole`.

Only System Administrators can create, edit, disable, delete, assign, or reset other users through **Settings → Users & Security**. The server protects the last enabled System Administrator. See [Manage users and application access](../admin/user-security.md).

On first start, setup is the only unauthenticated administrative flow. `GET /api/setup/status` returns `required`, `expiresAt`, `legacyReset`, and the fixed `username` when a legacy reset is required. `POST /api/setup/complete` accepts `code`, `username`, `displayName`, and `password`; it creates or repairs the first administrator, assigns `FW_SystemAdminRole`, and signs in that user. The code expires after 15 minutes or ten failures. Do not expose server logs containing the code to untrusted users.

## Server-side boundary

Every read and write path evaluates the authenticated session and authorization policy, including generic data APIs, import/export, named Functions, Reports, Views, Designer endpoints, and maintenance endpoints. Never rely on hidden routes, filtered menus, disabled buttons, or client-provided role information.

Security and credential tables are blocked from generic Data APIs and import/export even for System Administrators: `FW_User`, `FW_UserRole`, `FW_AppAccess`, `FW_Session`, `FW_WebArtifact`, `FW_Migration`, `FW_ViewToken`, and `FW_ViewTokenScope`. Use dedicated administration endpoints instead. Password hashes, plaintext passwords, and token hashes must not enter metadata responses, record responses, exports, audit data, or client state.

Use secure cookies behind HTTPS, protect first-run setup logs, keep updater tokens and integration secret keys outside source control, and give Docker socket access only to the dedicated updater.

## Migration and compatibility risks

- Existing users whose permissions depended on broad `FW_FrameworkUser` access lose that bypass. Grant only the App-scoped access and object privileges their work requires.
- An account named `admin` no longer bypasses authorization. Assign `FW_SystemAdminRole` explicitly when unrestricted framework access is required.
- Privileges created before View permissions were introduced need an explicit review. Source-table read permission does not automatically imply View access, and View access does not replace source-table read permission.
- Custom menus may disappear when all of their children become unauthorized. This is expected; verify menu targets after importing metadata.
- Clients that assume every metadata response contains every table, action, or report must handle filtered collections and disabled operations.
- Test upgrades against a copy of the database, including legacy role assignments and existing `FW_AppAccess` rows, before production rollout.

## Verification checklist

Test at minimum: no role/no App Access, Role only, App only, Customize only, App plus matching Role, and System Administrator. Cover direct API denial for tables, Functions, Reports, and Views; last-admin protection; password change/reset and session revocation; hidden empty menus; and immediate effect of Role/App Access changes.

## Related topics

[User administration](../admin/user-security.md) · [Views and Charts](views-and-charts.md) · [Configuration](../admin/configuration.md) · [Testing](testing.md)
