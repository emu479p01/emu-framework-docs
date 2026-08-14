# Manage application data

## Purpose

Export, replace, or permanently delete every business record owned by one App without changing its metadata.

## Audience

Framework administrators. These operations require `FW_SystemAdminRole`; App-level Designer permission is not sufficient.

## Export an App

1. Open **Settings → App Data Management**.
2. Find the App and review its table and row counts.
3. Select **Export data** and store the `.emuappdata` file outside the application host.

The package contains a manifest plus one payload per App-owned table. It preserves record IDs and records the framework version, schema fingerprints, row counts, and checksums.

## Replace an App's data

1. Export the current App data as a recovery copy.
2. Select **Replace data** and upload the matching `.emuappdata` package.
3. Review the preview, table counts, validation result, and warnings.
4. Type the App name exactly and confirm replacement.
5. Verify key records, references, totals, and user workflows.

Validation rejects a package for another App, incompatible framework data, checksum or schema mismatches, unknown tables or fields, duplicate IDs, and invalid cross-App references. Replacement deletes and reinserts all rows owned by the App in one transaction. It preserves IDs and bypasses per-record hooks, so run any required reconciliation explicitly after the operation.

## Delete all App data

Select **Delete all data**, review the affected tables, type the App name exactly, and confirm. This permanently removes App-owned business rows but does not delete the App, Models, metadata, or tables. Export a recovery package first.

Every export, replace, and delete is recorded in the App-data audit log. Do not use App Data Management as a substitute for a Full backup: users, security, Designer metadata, fonts, and integration configuration belong to other backup components.

## Related topics

[Backup](backup.md) · [Restore](restore.md) · [Database storage](database-storage.md) · [Security](../developer/security.md)
