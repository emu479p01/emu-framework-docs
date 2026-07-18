# Sign in and navigate

## Purpose

Open EmuFramework, sign in, and find an application page.

## Audience

Framework users.

## Prerequisites

An application URL and an account provisioned by an administrator.

## Procedure

1. Open the URL supplied by your administrator.
2. Enter your username and password.
3. Choose an app and menu item from the sidebar.
4. Use the sidebar button to collapse navigation on a small screen.
5. To change your password, open **My Account → Change Password**, enter the current password and a new password of at least 12 characters, then submit. Your old sessions are revoked and the current browser session is rotated.

On small screens, action pickers, line grids, imports, and administrative table views use stacked record cards or scrollable content instead of wide desktop tables. Buttons are enlarged for touch use. In Report Designer, swipe horizontally inside the canvas area to reach the full report width.

## Expected result

Only Apps that have `canOpen=true` and contain an object permitted by your assigned Roles are visible. Designer access is separate and may be available for a customized App even when that account cannot open business data.

## Common errors

- **Invalid credentials:** check the username or ask an administrator to reset access.
- **Missing menu:** ask an administrator to check both `canOpen` App Access and the matching Role/Duty/Privilege.

## Related topics

[Web Designer](web-designer.md) · [Security](../developer/security.md) · [User administration](../admin/user-security.md)
