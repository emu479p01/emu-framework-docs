# Define Data Entities

## Purpose

Describe a business document, a header record with its lines, as a Data Entity so users can exchange whole documents as spreadsheets and administrators can archive old documents safely.

## Audience

Application developers, ISV developers, and customizers who expose document-level import, export, or archiving.

## Prerequisites

The header Table, the line Tables, and a reference field on each line Table that points to the header must already exist. Read [Work with metadata](metadata.md) and [Extensions](extensions.md). Data Entities were introduced in v1.1.0 and `dataEntityExtension` in v1.2.0.

## Concepts

A Data Entity does not create tables. It names a **root table**, the fields exchanged for it, a **business key** that identifies one document, and optional **lines**. Import, export, and archive all read the same definition, and each document is treated as one unit.

| Term | Meaning |
| --- | --- |
| Header | One record of the root table. |
| Business key | The root fields that uniquely identify a document, such as `orderNo`. Import matches headers on it and archive uses it to identify the document. |
| Line | A child table that references the root table. |
| Line key | The line fields that identify one line within its header. |

## Data Entity artifact

`dataEntity` is a base kind, so a higher Layer can replace it. `rootTable`, `businessKey`, and a non-empty `fields` are required.

```json
{
  "kind": "dataEntity",
  "name": "SALES_OrderEntity",
  "app": "sales",
  "model": "Core",
  "layer": "ISV",
  "label": "Sales orders",
  "rootTable": "SALES_Order",
  "businessKey": ["orderNo"],
  "fields": ["orderNo", "orderDate", "customerId", "amount"],
  "lines": [
    {
      "name": "Lines",
      "table": "SALES_OrderLine",
      "parentReference": "orderId",
      "fields": ["lineNo", "itemCode", "quantity", "unitPrice"],
      "lineKeys": ["lineNo"]
    }
  ],
  "archiveEligible": true,
  "businessDateField": "orderDate"
}
```

| Property | Required | Rule |
| --- | --- | --- |
| `rootTable` | Yes | Must be an existing Table. |
| `businessKey` | Yes | Non-empty. Every field must exist on the root table (or be `id` or a system field) and must also appear in `fields`. |
| `fields` | Yes | Non-empty root fields that are exchanged. |
| `lines[].name` | Yes | Identifies the line. It becomes the sheet name or CSV source name. |
| `lines[].table` | Yes | Existing line Table. |
| `lines[].parentReference` | Yes | A `reference` field on the line Table that points at `rootTable`. |
| `lines[].fields` | Yes | Non-empty line fields that are exchanged. |
| `lines[].lineKeys` | Yes | Non-empty. Every key must also appear in the line's `fields`. |
| `archiveEligible` | No | Opts the entity in to archiving. Requires `businessDateField`. |
| `businessDateField` | No | A root field of type `date` or `datetime`. |

Registry validation rejects unknown tables and fields, a business key or line key that is not listed in `fields`, a `parentReference` that does not reference the root table, and `archiveEligible` without a date field. Each message names the Data Entity and the offending field. File-based Apps keep entities in `dataEntities` and extensions in `dataEntityExtensions`.

Add a `dataEntity.<name>.label` resource to a Translation to localize the entity label; see [Localize metadata with Translations](localization.md).

## Import and export

All routes use the signed-in user's session, and the entity is read from the merged registry.

| Route | Purpose |
| --- | --- |
| `GET /api/data-entities/:name/export` | Download XLSX. Add `?format=csv` for a CSV ZIP package. |
| `POST /api/data-entities/:name/import/preview` | Upload a file as multipart data; the server stages it and returns a `previewId`. |
| `POST /api/data-entities/:name/import/commit` | Commit a staged preview with `{ "previewId": "..." }`. |
| `GET /api/data-entities/jobs/:jobId/errors` | Download the error file of your own import job. |

### Permissions

Export needs `read` on the root table and every line table. Import needs both `create` and `update` on all of them. A missing permission returns `403` with `Access denied: Data Entity read|write on '<table>'`. While a licensed Model that owns the entity is read-only, each imported document fails and archive runs are refused; see [Model deployment and ISV licenses](model-deployment.md).

### File formats

XLSX has a `Header` sheet (the first sheet is used if there is none) and one sheet per line, named after `lines[].name`. Row 1 contains column names. Header columns are the entity `fields`. Every line sheet starts with the business-key columns, which tie each row to its header, followed by the line `fields`. The `parentReference` column is not exchanged; the server fills it in.

A CSV package is a ZIP file. It contains one UTF-8 CSV file per source (`Header.csv`, `Lines.csv`, ...) and a `manifest.json`:

```json
{
  "format": "emuframework-data-entity",
  "schemaVersion": 1,
  "entity": "SALES_OrderEntity",
  "exportedAt": "2026-10-08T03:15:00.000Z",
  "sources": [
    { "name": "Header", "file": "Header.csv" },
    { "name": "Lines", "file": "Lines.csv" }
  ]
}
```

Import rejects a package whose `format`, `schemaVersion`, or `entity` do not match, a source that is not listed, and a file path containing `..` or a backslash. Legacy `.xls` returns `415` (`Legacy .xls files are not supported; use .xlsx`), and any other extension also returns `415`. Uploads are limited to 256 MiB. Export is capped at 50,000 headers. Encrypted fields are never written to an export.

### Import behavior

1. **Preview** parses the file and stages it in durable job storage (`EMU_DATA_JOB_PATH`, default a `data-jobs` directory beside the data database). The response contains `previewId`, `entity`, the number of `documents`, row counts for each source, and a ten-row `sample`. Nothing is written to business tables.
2. **Commit** processes every header as its own document. Only the user who created the preview can commit it, and only once; otherwise the server returns `410 Import preview is unavailable`.
3. Each document is **atomic**. The header and all its lines are written in one transaction, so a failure rolls back that document and the import continues with the next one.
4. A header is **upserted** by business key: an existing header is updated and a new one is created. A line is matched by `parentReference` plus its line keys, then updated or created. Lines already stored but absent from the file are left alone.
5. Writes go through normal `DataContext` rules, so defaults, `initValue`, `validateWrite`, events, and permissions run. Fields that are read-only, or not editable for the operation (`allowEditOnCreate`, `allowEdit`), are skipped.

The commit response reports the result and links the error file when any document failed:

```json
{
  "jobId": "5b4f6c0a-3d0e-4b0e-9a3f-2f1c0f6d9e21",
  "inserted": 12,
  "updated": 3,
  "linesInserted": 40,
  "linesUpdated": 9,
  "failed": 1,
  "errorFile": "/api/data-entities/jobs/5b4f6c0a-3d0e-4b0e-9a3f-2f1c0f6d9e21/errors"
}
```

The error file is CSV with the columns `document` (1-based row in the Header source), `key` (for example `orderNo=SO-1001`), and `error`. A blank business key fails with `Header business key '<field>' is required`, and a blank line key fails with `<line> line key '<field>' is required`.

## Extend a Data Entity

`dataEntityExtension` adds to an existing Data Entity without replacing it. It can append root fields, add whole new lines, or append fields to an existing line, and nothing else. The target property is `dataEntity`.

```json
{
  "kind": "dataEntityExtension",
  "name": "SALES_Customizations_SALES_OrderEntity_Extension",
  "app": "sales",
  "model": "Customizations",
  "layer": "CUS",
  "dataEntity": "SALES_OrderEntity",
  "fields": ["customerReference"],
  "lineExtensions": [
    { "name": "Lines", "fields": ["remark"] }
  ]
}
```

| Property | Rule |
| --- | --- |
| `fields` | Appended to the root fields. A field already in the entity is rejected: `field '<f>' already exists on '<entity>'`. |
| `lines` | New lines, with the same shape as `dataEntity.lines`. A name that already exists is rejected. |
| `lineExtensions` | `name` of an existing line plus `fields` to append. An unknown line or an existing field is rejected. |

Each collection, when present, must be non-empty. The extension schema does not accept `rootTable`, `businessKey`, a line's `table`, `parentReference`, or `lineKeys` of an existing line, or archive settings, so they cannot change. The fields must exist on the Table, so add them through a `table` or `tableExtension` first. Normal Extension rules apply: the Layer must be higher than the target, the target App must be a declared dependency, and one extension per `(app, model, kind, target)`. See [Create extensions](extensions.md).

Import, export, archive, and restore all use the merged entity. An archive created before an extension can still be restored. Per the v1.2.0 release notes, fields that did not exist at archive time follow the import defaults, and a new mandatory field without a default fails with a clear error and keeps the archive.

## Archive eligibility

Archiving moves old documents out of the live database into immutable files. It is **opt-in per entity**: only an entity with `archiveEligible: true` and a `businessDateField` can have an archive policy. A system administrator creates the policy; it is not metadata.

- The policy's business date must be one of the entity's exchanged root `fields` and of type `date` or `datetime`. Otherwise the API returns `422`.
- A run selects headers whose date is on or before the cutoff (now minus `ageDays`), oldest first, up to `batchSize`.
- Policy defaults are `ageDays` 365, `batchSize` 100 (1 to 10,000), `schedule` `weekly` (or `daily`), `weekday` 0 to 6, `timezone` `Asia/Bangkok`, and `includeAttachments` true. The scheduler wakes hourly and cannot be pinned to an exact time.
- Each document is written to a checksummed JSON payload (SHA-256) and cataloged before the live header, lines, and attachments are deleted in one transaction. A document that already has an archive, or that has attachments when the policy excludes them, is skipped with an error in the job summary.
- Restore re-creates the document through the same atomic import path. It skips a document whose business key already exists in live data (`409`) and restores attachments with new identifiers.
- Archived data is retained indefinitely. Automatic purge is not provided.

Choose `businessDateField` for the date after which a document is closed, such as a posting date. Do not mark an entity eligible if later processes still expect the live records.

## Procedure

1. Define the header and line Tables, with a reference field from each line to the header.
2. Create the `dataEntity` and choose a business key that is genuinely unique per document.
3. Export a few documents, edit them, and import them back to confirm the round trip.
4. Test a bad row in the middle of a file and confirm that only that document fails and the error file names it.
5. Test with a user who has only read permission to confirm that import returns `403`.
6. Mark the entity `archiveEligible` only after you test restore into an empty database.

## Related topics

[Storage and archive administration](../admin/storage-and-archive.md) · [Artifact kinds](artifact-types.md) · [Extensions](extensions.md) · [Record lifecycle](record-lifecycle.md) · [Attachments](attachments.md) · [Security](security.md)
