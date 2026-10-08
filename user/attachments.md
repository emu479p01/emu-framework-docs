# Work with attachments

## Purpose

Attach files, notes, and web links to records, preview images, and give a Function photos from your camera.

## Audience

Framework users.

## Prerequisites

The record must already be saved. You need read permission on the record to see its attachments, and update permission to add or delete them.

## Add an attachment to a record

1. Open a saved record.
2. In the **Attachments** section, choose what to add:
   - **Upload file** to pick a file from your device.
   - Type in the **Add a note** box and select **Add note**.
   - Type a web address in the `https://…` box and select **Add URL**. Only `http` and `https` addresses are accepted.
3. The new attachment appears in the list.

To remove an attachment, select **Delete** on its row and confirm **Delete this attachment?**. Deleting changes the record, so it needs update permission.

## Attach files to a line

On a form with a line grid, select **Files** on the row. A **Line attachments** dialog opens with the same controls. The row must be saved first.

## What you can upload

An administrator sets the limits. By default a file can be up to 25 MB and can be a PDF, Word (`.docx`), Excel (`.xlsx`), PowerPoint (`.pptx`), CSV, plain text, PNG, JPEG, or WebP file. The file extension must match the file type. If an upload is refused, the message names the cause: **Attachment exceeds the configured size limit** or **Attachment type ... is not allowed**. Ask your administrator if you need a larger limit or another type.

## Preview images

PNG, JPEG, and WebP attachments show a thumbnail. Select the thumbnail to open the image in a viewer, where you can also choose **Download** or **Close**. On a desktop, press Escape to close the viewer and return to where you were. On a phone, the image fits the screen, touch targets are large, and you can still pinch to zoom. If an image cannot be loaded, the viewer shows **The image could not be loaded**.

Other files, notes, and links keep their normal display: select a file to download it, and a link opens the web address.

## Give a Function photos

Some Functions ask for images, for example to scan a receipt for an order. When you run one, its dialog shows the record number and two choices:

- **Choose images** to pick image files (JPEG, PNG, or WebP).
- **Take a photo** to use the device camera, on devices that support it.

Selected images appear with a small preview, and you can remove one before continuing. The images are uploaded when you confirm the dialog and become attachments of the record. If one upload fails, the ones that succeeded stay and you can retry only the failed file. Retrying does not create duplicates. Images stay attached even if the Function itself fails afterwards. A Function that accepts one image rejects a second. The Function must run against a saved record, and you need permission to run it and to update that record.

## Expected result

Attachments appear on the record, can be downloaded or previewed by anyone who can read the record, and move with the record if an archive policy that includes attachments archives and later restores it.

## Common errors

- **The Attachments section is missing:** the record is not saved yet, or you lack read permission.
- **Upload is refused:** the file is too large or its type is not allowed.
- **Delete or upload is denied:** you do not have update permission on the record.
- **Image does not match its file type:** the file extension and content disagree, for example a renamed file. Export the image again.

## Related topics

[Sign in and navigate](getting-started.md) · [Manage storage and archiving](../admin/storage-and-archive.md) · [Attachments and Function image input](../developer/attachments.md)
