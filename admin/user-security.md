# Manage users and application access

## Purpose

Provision users with deny-by-default access, assign Roles and App Access independently, reset passwords, and protect the last System Administrator.

## Audience

System Administrators.

## Open Users & Security

Sign in with an enabled account that holds `FW_SystemAdminRole`, then open **Settings → Users & Security** (`/system/security/users`). The page uses dedicated administration endpoints; security tables are not editable through generic forms or `/api/data/FW_*`.

## Create a user

1. Select **New user**.
2. Enter the immutable username, display name, and an initial password of at least 12 characters.
3. Select existing Roles from the list. Leave Roles empty when the account should not access business objects.
4. For each existing business App, select **Open**, **Customize**, both, or neither.
5. Save and test the account with the permission matrix in [Understand security](../developer/security.md).

The server validates Role and App names. It does not accept free-text assignments. A new user with no Role and no App Access cannot see or call a business App.

## Choose App Access

| Grant | Result |
| --- | --- |
| `canOpen=true` | Allows the App to pass the first runtime gate; object privileges are still required. |
| `canCustomize=true` | Allows Designer access to this App only; it grants no App entry or business-data access. |
| Both | Allows design plus the possibility of runtime access when a Role grants the object. |
| Neither/no row | Denies both capabilities for the App. |

Grant `canCustomize` only to people who design that App. Do not use `FW_FrameworkUser` as a Designer bypass; in v0.1.1.0 it is a legacy marker only.

## Change or reset a password

Every signed-in user can open **My Account → Change Password** and submit the current password plus a new password of at least 12 characters. The server revokes the user's old sessions and rotates the current session so the user can continue working.

A System Administrator can select a user and choose **Reset password**. The supplied password works immediately; the user is not forced through a first-login change. Resetting revokes every existing session for that account.

Passwords and hashes are never returned in user, metadata, export, or audit responses. Never send passwords through generic record APIs.

## Disable or delete a user

Disabling a user revokes their sessions. Deleting removes the user and their Role/App Access assignments. The server rejects any disable, delete, or Role removal that would leave no enabled account with `FW_SystemAdminRole`.

## Dedicated APIs

```text
GET    /api/system/security/catalog
GET    /api/system/security/users
POST   /api/system/security/users
PATCH  /api/system/security/users/:id
DELETE /api/system/security/users/:id
POST   /api/system/security/users/:id/reset-password
POST   /api/account/change-password
```

System Administrator authority is determined only by the `FW_SystemAdminRole` assignment. A username such as `admin` has no implicit privilege.

## Related topics

[Security model](../developer/security.md) · [Power BI View tokens](power-bi-view-api.md) · [Sign in and navigate](../user/getting-started.md)
