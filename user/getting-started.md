# Sign in and navigate

## Purpose

Open EmuFramework, sign in, find an application page, and learn how navigation, language, and new records behave.

## Audience

Framework users.

## Prerequisites

An application URL and an account provisioned by an administrator.

## Procedure

1. Open the URL supplied by your administrator.
2. Enter your username and password.
3. Choose an app and menu item from the sidebar.
4. Use the sidebar button to collapse navigation on a small screen.
5. To change the display language, open the user menu and choose a language. See [Change the language](#change-the-language).
6. To change your password, open **My Account → Change Password**, enter the current password and a new password of at least 12 characters, then submit. Your old sessions are revoked and the current browser session is rotated.

On desktop, expanding an App branch collapses the previously open branch. The browser remembers the last open branch for the current session. On mobile, navigation remains an overlay designed for touch.

On small screens, form controls use a 16px input size to prevent automatic iPhone Safari zoom while preserving manual pinch-to-zoom. Action pickers, line grids, imports, and administrative table views use stacked record cards or scrollable content instead of wide desktop tables. In Report Designer, swipe horizontally inside the canvas area to reach the full report width.

## Reopen pages from Recent

Each App has a **Recent** entry at the top of its menu. It lists the ten menu items you opened most recently in that App, with the latest first. Opening an item again moves it to the top, and an item never appears twice. Recent is separate for each App, so another App's history does not push your items out. **Settings** keeps its own Recent list. If you have not opened anything yet, Recent shows **No recently opened items**.

Recent replaced Favorites in 1.2.0, so there are no star buttons. Recording history never delays opening a page.

## Change the language

Open the user menu and choose a language from the list; the current language has a check mark. Framework screens are available in English and Thai, and an App can also translate its own labels. Your choice is saved with your account. The page keeps your place, the open record, and any unsaved form values while the texts change. If the language is saved but the texts fail to reload, a message offers **Retry**.

Values that people enter, such as names, descriptions, and notes, are never translated.

## Create a new record

1. Open a list and select **New**.
2. The form opens with default values already filled in. Values that the system assigns, such as a document number, are shown immediately, before you save, and are read-only.
3. Fill in the fields and select **Save**. The record is created and the page switches to the saved record.

Nothing is stored until you save, so leaving a new form without saving does not use up a record. A new form is valid for 24 hours. If saving fails with a `409` message such as **Draft expired or unavailable** or **Metadata changed**, the form is out of date. Open **New** again and re-enter your values; the values you typed stay on the page, but they are not carried over automatically. A metadata change happens when an administrator or customizer updates the App while your form is open.

## Dates, times, and numbers

Dates and times with a time component are stored in UTC and shown in your browser's time zone, so two users in different time zones see the same moment at their own local time. Numbers show with thousands separators. Codes, IDs, references, and text are shown exactly as stored.

## Attachments

Saved records can carry files, notes, and web links. Open a record and use the **Attachments** section; on line grids, use **Files** on a row. See [Work with attachments](attachments.md).

## Expected result

Only Apps that have `canOpen=true` and contain an object permitted by your assigned Roles are visible. Designer access is separate and may be available for a customized App even when that account cannot open business data.

## Common errors

- **Invalid credentials:** check the username or ask an administrator to reset access.
- **Missing menu:** ask an administrator to check both `canOpen` App Access and the matching Role/Duty/Privilege.

## Related topics

[Work with attachments](attachments.md) · [Web Designer](web-designer.md) · [Security](../developer/security.md) · [User administration](../admin/user-security.md)
