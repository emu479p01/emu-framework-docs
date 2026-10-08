# Understand the record lifecycle

## Purpose

Explain how a new record is created through a draft, how audit fields behave, why lifecycle handlers must be synchronous, and how datetimes and numbers are handled. These behaviors changed in v1.4.0.

## Audience

Application developers who write `initValue`, `validateWrite`, or data event logic, call the generic data API, or build Views and Reports over audit data.

## Prerequisites

Read [Use hooks and data events](hooks-events.md) and [Work with metadata](metadata.md). Sign-in cookies are assumed for every endpoint on this page.

## Create a record through a draft

Opening **New** on a generated Form no longer inserts a row. The client asks the server for a draft, shows it, and inserts the record only when the user saves.

```http
POST /api/data/SALES_Order/drafts
Content-Type: application/json

{}
```

The optional body can supply initial values for writable fields. The server checks `create` permission, runs field defaults and every `initValue` hook inside a transaction, and stores the result. It returns the draft token, its expiry, and the initial record:

```json
{
  "token": "6f0a3c1e-1c5e-4d0b-bd0c-1f7c2f8e7a90",
  "expiresAt": "2026-10-09T03:15:00.000Z",
  "record": { "id": null, "orderNo": "SO-00001", "status": 0 }
}
```

Values that `initValue` sets, including read-only fields such as a document number, are visible to the user straight away, and no row exists yet. Saving sends only what the user edited:

```http
POST /api/data/SALES_Order/drafts/6f0a3c1e-1c5e-4d0b-bd0c-1f7c2f8e7a90/save
Content-Type: application/json

{ "customerId": 12, "note": "Rush order" }
```

A successful save returns `201` with the stored record. The server merges the user's values over the saved snapshot, then validates and inserts the record through the normal write path, so `validateWrite`, data events, and permissions all run. The `initValue` hooks do **not** run a second time.

### Draft rules

- A draft is stored encrypted in `FW_RecordDraft` in the data database. It is bound to the creating user and Table, expires after 24 hours, and survives a server restart. Expired drafts are deleted when any new draft is created.
- The save is atomic. If validation fails or an event throws, the insert rolls back and the draft stays usable, so the user can correct the values and save again.
- Saving the same token twice returns the record that the first save created instead of inserting again, which makes client retries safe.
- A missing, expired, or foreign token returns `409` with `Draft expired or unavailable. Open a new draft; your entered values have not been saved.`
- A draft created before a change to Tables, Scripts, Functions, or Model artifacts returns `409` with `Metadata changed. Open a new draft before saving; keep your entered values.` Reopen the draft and re-enter the values. The generated client keeps what the user typed in the page.
- Read-only fields cannot be written from the request body; the server rejects them with `422` (`<table>.<field> cannot be edited during creation`). Audit columns in the body are ignored.
- `create` permission and any license guard are checked both when the draft is created and when it is saved.
- Encrypted fields appear in responses as the mask `••••••••`, never as the stored value.

`POST /api/data/:table` still creates a record in a single request. It runs `initValue`, applies the body, and inserts. Use it for integrations that do not show a form first.

### Number sequences and drafts

Because a draft is not a row, a value produced by `initValue` is not reserved. Two users who open drafts at the same time can receive the same document number. Protect such numbers with a unique index, and allocate gap-free numbers in `validateWrite` or `onInserting`, where the write is already in progress.

```js
kernel.hooks.register('SALES_Order', {
  initValue(record, ctx) {
    const last = ctx.select('SALES_Order').orderBy('id', 'desc').firstOnly();
    record.set('orderNo', 'SO-' + String((last ? last.id : 0) + 1).padStart(5, '0'));
  }
});
```

### Read-only mandatory fields

`initValue` can fill read-only fields, such as a document number, on the draft, and the user sees the value before saving. Metadata validation in v1.4.0 still rejects `mandatory` together with `readOnly` (`Read-only field '<name>' cannot be required`), and load-time normalization and the `tableExtension` merge remove `mandatory` from a read-only field. Declare such a field as `readOnly` only, fill it in `initValue`, and enforce presence in `validateWrite` when the value is essential.

## Audit fields

Every Table has the physical columns `createdAt`, `createdBy`, `modifiedAt`, and `modifiedBy`. v1.4.0 also exposes four read-only virtual aliases that map to them. No new columns are added.

| Alias | Maps to | Type |
| --- | --- | --- |
| `sys_createdBy` | `createdBy` | string |
| `sys_createdAt` | `createdAt` | datetime |
| `sys_modifiedBy` | `modifiedBy` | string |
| `sys_modifiedAt` | `modifiedAt` | datetime |

Use the aliases wherever a field name is accepted for reading:

- List filters and sorting: `GET /api/data/SALES_Order?filter.sys_createdBy=admin&sort=sys_createdAt`.
- `Query.where`, `search`, and `orderBy`, and `Record.get` and `set` in server code.
- View columns, filters, and joins as `alias.sys_createdBy`.
- Report field elements and Tablix columns.

```json
{ "name": "creator", "expression": { "type": "field", "ref": "o.sys_createdBy" } }
```

`GET /api/metadata` now lists the system fields in each Table's `fields`: `id`, the four audit columns, and the four aliases, each marked `readOnly`. Clients that iterate `fields` must handle them.

Rules the server enforces:

- Updates keep the original `createdAt` and `createdBy`. The server sets `modifiedAt` and `modifiedBy` itself. A request body that includes any system or alias name has those entries ignored.
- System field names are reserved case-insensitively. A declared field named `CreatedAt` or `SYS_modifiedBy` fails registry validation with `<table>.<field>: '<field>' is a reserved system field`.
- If a stored table already has a physical column that matches an alias, schema synchronization stops before it changes anything and reports `<table>: reserved audit aliases collide with stored columns: <columns>`. Rename or migrate the column, then restart.
- Encrypted fields cannot be filtered, searched, sorted, or used in a View; the server rejects them with `encrypted fields cannot be filtered, searched, or sorted`.

## Lifecycle handlers must be synchronous

`initValue`, `validateWrite`, `validateDelete`, and every data event handler run inside the database transaction. They must return without awaiting.

```js
// Rejected: registering an async function
kernel.hooks.register('SALES_Order', {
  async validateWrite(record, ctx) { /* ... */ }
});
```

The framework raises `ValidationError` with `<table>.<hook>: async lifecycle handlers are not supported` in these cases:

- Registering an `async` function as a hook or event handler fails at registration. A Script that does so fails to load and is reported against that Script.
- A handler that returns a Promise fails when it runs, and the write is rolled back. Over REST this is a `422`.
- Passing an `async` callback to `ctx.tts()` fails with `Transaction: async lifecycle handlers are not supported`.

This is a breaking change in v1.4.0. Earlier releases ran such code outside the transaction. Review every Script and `scriptExtension` before you upgrade. Move awaited work, such as HTTP calls, email, or file access, into a Function with `executionMode: "async"`, and write to the database from it inside short synchronous `ctx.tts()` blocks. See [Develop Functions and actions](functions.md).

## Datetime and number values

- A `datetime` value is stored and returned as a UTC ISO string, for example `2026-08-15T15:52:39.000Z`.
- Input that has an offset is converted to UTC. Input without an offset, including values stored by older releases, is read as UTC. A space between date and time is accepted. An unparsable value fails with `<table>.<field>: invalid datetime`.
- The generated client shows datetimes in the browser time zone and sends UTC back. Business logic should compare and store UTC and convert only for display.
- A `date` field is a plain date string and is not converted.
- The client displays numbers with grouping separators. IDs, references, and strings are never reformatted. The API always returns numbers as JSON numbers.

## Procedure

1. Search your Scripts for `async` hooks and event handlers and move their awaited work into an async Function.
2. Decide which values the user needs to see before saving, and set them in `initValue`.
3. Test a draft that expires, one opened before a metadata change, one saved twice, and one saved by a different user.
4. Test a failed save and then a corrected save of the same draft.
5. Replace `createdBy` and `createdAt` in Views and filters with the aliases where you want a stable contract.
6. Send datetimes with and without an offset and confirm the stored UTC value.

## Related topics

[Hooks and data events](hooks-events.md) · [Scripts](scripts.md) · [Functions and actions](functions.md) · [Views and Charts](views-and-charts.md) · [Attachments](attachments.md) · [Security](security.md)
