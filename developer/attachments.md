# Work with record attachments

## Purpose

Attach files, notes, and web links to saved records, preview images, and understand which limits and permissions the server enforces.

## Audience

Application developers who integrate with the attachment endpoints, add image Functions, or reason about attachment security.

## Prerequisites

A business Table with saved records. Attachments were introduced in v1.1.0, previews in v1.2.0, and WebP preview and Function image input in v1.4.0. Administrators handle storage volumes and backups; see [Configuration](../admin/configuration.md).

## Concepts

An attachment belongs to one saved record of a business Table, either a Form header or a Line row. Records that have not been saved yet cannot have attachments. Framework tables (`FW_*`) cannot be a parent, and the server returns `404` for them.

| Kind | Content |
| --- | --- |
| `file` | An uploaded file. Content is stored on the file volume under an opaque storage key, with its size and SHA-256 recorded. |
| `note` | Plain text. The text is required. |
| `url` | An `http:` or `https:` link. |

Permissions come from the parent record, not from the attachment: reading, downloading, and previewing need `read` on the parent Table, and uploading, adding a note or link, or deleting needs `update`. There is no separate attachment privilege. A licensed Model that is read-only also blocks attachment changes; see [Model deployment and ISV licenses](model-deployment.md).

## Endpoints

| Method and path | Needs | Purpose |
| --- | --- | --- |
| `GET /api/attachments/:table/:id` | `read` | List the record's attachments, newest first. |
| `POST /api/attachments/:table/:id/file` | `update` | Upload one file as multipart data. |
| `POST /api/attachments/:table/:id/note` | `update` | Add a note with `{ "name", "text" }`. |
| `POST /api/attachments/:table/:id/url` | `update` | Add a link with `{ "name", "url" }`. |
| `GET /api/attachments/:attachmentId/download` | `read` | Stream the file as a download. |
| `GET /api/attachments/:attachmentId/preview` | `read` | Stream an image inline. |
| `DELETE /api/attachments/:attachmentId` | `update` | Remove an attachment and its content when no other attachment shares it. |

The list returns `{ "items": [...] }`. Each item has `id`, `kind`, `name`, `createdAt`, and `createdBy`, plus `text` for notes, `url` for links, and `mimeType`, `bytes`, and `sha256` for files.

```json
{
  "items": [
    {
      "id": "0d7a7a9e-8a88-4f0d-8f0e-5d3a8d1b6c11",
      "kind": "file",
      "name": "receipt.png",
      "mimeType": "image/png",
      "bytes": 48213,
      "sha256": "3f2a...e91c",
      "createdAt": "2026-10-08T03:15:00.000Z",
      "createdBy": "admin"
    }
  ]
}
```

Uploads, notes, and links return `201` with the new `id`. A record id that is not a positive integer returns `400`, and an unknown record returns `404`. Deleting a record through `DELETE /api/data/:table/:id` also removes its attachments.

## Limits and allowed types

Files are streamed to disk while the size and hash are computed, so large uploads do not load into memory.

| Limit | Behavior |
| --- | --- |
| Size | 25 MiB by default. `EMU_ATTACHMENT_MAX_BYTES` changes it. Larger files return `413` (`Attachment exceeds the configured size limit`) and leave nothing behind. |
| Type | The file extension and the declared MIME type must agree and must be allowed, otherwise `415` (`Attachment type '<ext>' is not allowed`). |
| Link | Only `http:` and `https:` URLs. Anything else returns `400`. |
| Name | A note or link name is trimmed to 255 characters. |

Built-in types are PDF (`.pdf`), Word (`.docx`), Excel (`.xlsx`), PowerPoint (`.pptx`), CSV (`.csv`), text (`.txt`), and images (`.png`, `.jpg`, `.jpeg`, `.webp`). Set `EMU_ATTACHMENT_ALLOWED_TYPES` to a comma-separated list of MIME types to narrow the allowed set. Legacy Office formats and archives are not accepted.

Storage lives under `EMU_FILE_STORAGE_PATH`. Docker deployments default to `/data/files` and should mount a separate `/files` volume. Interrupted uploads and files without a catalog row are cleaned up at start-up.

## Preview images

`GET /api/attachments/:attachmentId/preview` serves PNG, JPEG, and WebP files inline. The server reads the file signature and refuses content that does not match the recorded type, so a renamed file is never rendered as an image. Responses carry `Content-Disposition: inline`, `X-Content-Type-Options: nosniff`, and `Cache-Control: private, no-store`.

| Status | Meaning |
| --- | --- |
| `409` | The attachment is a note or link, not a file. |
| `410` | The content is missing from the volume; contact an administrator. |
| `415` | The file is not a PNG, JPEG, or WebP image, or its content does not match its type. |

Other types, including PDF and Office files, are available only through `download`, which always sends `Content-Disposition: attachment`.

## Images for Functions

A Function can declare `imageInput` so users select photos before it runs. The client uploads each image to the same file endpoint with two query parameters:

```http
POST /api/attachments/SALES_Order/42/file?function=SALES_ScanReceipt&uploadId=<uuid>
```

`function` must name a Function whose `imageInput.table` is the parent Table, the caller must be allowed to run it, and the file must be a JPEG, PNG, or WebP whose signature matches. `uploadId` is a client-generated UUID that becomes the attachment id. Repeating a request with the same `uploadId` returns the existing attachment instead of storing it again, which lets a client retry one failed file safely. The same ID from another user or record is rejected. See [Develop Functions and actions](functions.md#image-input) for the metadata and the arguments the Function receives.

## Security considerations

Treat uploaded content as untrusted. Keep the type allow list as narrow as the application needs, keep the file volume outside any public web root, and test with a user who can read but not update the parent record. Attachment content is not scanned or translated, and it is reachable only through the endpoints above.

## Testing

Test upload, list, download, preview, and delete as users with and without `read` and `update` on the parent Table. Add a file over the size limit, a disallowed extension, a mismatched extension and MIME type, a renamed non-image sent to `preview`, and a repeated `uploadId`. Delete the parent record and confirm the attachments are gone.

## Related topics

[Using attachments](../user/attachments.md) · [Storage and archive administration](../admin/storage-and-archive.md) · [Develop Functions and actions](functions.md) · [Record lifecycle](record-lifecycle.md) · [Data Entities](data-entities.md) · [Reports](reports.md) · [Security](security.md)
