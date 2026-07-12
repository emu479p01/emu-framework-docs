# EmuFramework Development Guide

This guide explains how to develop applications and framework extensions with EmuFramework. It is based on the current implementation in the [EmuFramework source repository](https://github.com/emu479p01/emu-framework), especially `@emu/core`, `@emu/server`, `@emu/client`, and `@emu/cli`.

## 1. EmuFramework Overview

EmuFramework is a metadata-driven TypeScript framework for business applications. An application is described by metadata artifacts such as tables, fields, forms, menus, security definitions, reports, scripts, and functions. The framework uses those artifacts to provide:

- SQLite database schema and data access.
- Generated list and form pages.
- Navigation and application menus.
- Role-based table and form permissions.
- Browser-based Web Designer editing.
- Import/export and PDF reporting.
- Server-side business logic through hooks, events, scripts, and named functions.

The important design principle is that every data operation should pass through the framework data layer. This ensures that validation, hooks, events, transactions, and authorization apply consistently to REST requests, scripts, tests, and internal services.

## 2. Repository Structure

| Location | Responsibility |
| --- | --- |
| `packages/core` | Metadata types and validation, registry, SQLite synchronization, security policy, data API, hooks, events, and kernel runtime. |
| `packages/server` | Fastify API, authentication, Designer API, generic data API, named action API, reports, import/export, and system services. |
| `packages/client` | Vue application shell, generated forms/lists, navigation, Web Designer, report designer, and client state. |
| `packages/cli` | Commands for creating apps, objects, modules, extensions, and workspace registrations. |
| `packages/mcp` | MCP catalog and AI/developer integration. |
| `emu-framework-docs` | User, administrator, and developer documentation maintained in this repository. |
| `scripts` | Maintenance, update, database, and version-checking scripts. |

The main runtime flow is:

```text
JSON metadata / Web Designer artifact
        ↓
Schema validation and MetadataRegistry
        ↓
Kernel registration and business database synchronization
        ↓
Generated API and Vue UI
        ↓
DataContext → validation → hooks/events → transaction → SQLite
```

## 3. Development Environment

### Prerequisites

The repository is configured for:

- Node.js `24.18.0` (the package engine also permits supported Node 22 versions).
- pnpm `11.12.0`.
- TypeScript 5.
- SQLite through `better-sqlite3`.

The Windows launcher can provision a portable runtime in `.tools`. For normal repository development:

```sh
pnpm install
pnpm dev
```

The development server runs on port `3399` and the Vite client runs on port `5199`, as described in [Developer setup](setup.md).

### Verification commands

```sh
pnpm check:versions
pnpm typecheck
pnpm test
pnpm build
docker compose config
```

Package-level commands are also available:

```sh
pnpm --filter @emu/core test
pnpm --filter @emu/server test
pnpm --filter @emu/client test
pnpm --filter @emu/core typecheck
```

Use development credentials only with local databases. The seeded account is `admin` / `admin`; change it before using real data.

## 4. Metadata-driven Architecture

### 4.1 Metadata artifacts

Metadata artifacts are discriminated by `kind` and identified by a stable `name`. Labels are user-facing; names are identifiers and should not be casually changed after deployment.

Supported base kinds include:

```text
app, enum, table, form, menu, privilege, duty, role,
script, function, report
```

Supported extension kinds include:

```text
tableExtension, enumExtension, formExtension, menuExtension,
privilegeExtension, dutyExtension, roleExtension, scriptExtension
```

The schema is defined in `packages/core/src/metadata/schema.ts` and the TypeScript contracts are defined in `packages/core/src/metadata/types.ts`.

### 4.2 Validation and registry

Metadata goes through two levels of validation:

1. JSON-schema validation checks the wire shape, property types, allowed values, and required properties.
2. `MetadataRegistry` performs cross-reference validation, such as checking that an enum, table, form, field, reference, report data source, or action target exists.

Use the framework validator before registering or applying artifacts:

```ts
import { validateMetadataArtifact } from '@emu/core';

const diagnostics = validateMetadataArtifact(artifact);
if (diagnostics.length > 0) {
  throw new Error(JSON.stringify(diagnostics));
}
```

Typical validation failures include:

- Invalid or empty artifact names.
- Unknown enum names.
- Unknown reference tables or fields.
- Unknown form fields or report bindings.
- Invalid form line references.
- Required references configured with `onDelete: "setNull"`.
- Duplicate artifact names.
- Missing app dependencies.

### 4.3 Layers and overrides

The layer order is:

```text
SYS < ISV < LOC < DEV < CUS
```

Higher layers override lower-layer base artifacts with the same logical identity. Extensions accumulate into their target instead of replacing it. Use an extension when a feature should add fields, menu items, permissions, or form behavior without copying and replacing the base artifact.

Recommended usage:

- `SYS`: framework-owned definitions.
- `ISV`: vendor or product definitions.
- `LOC`: localization or site-specific definitions.
- `DEV`: development customizations.
- `CUS`: customer customizations.

### 4.4 Database synchronization

Table metadata drives SQLite schema creation and additive synchronization. Adding tables, fields, and indexes is supported by the normal synchronization path. Removing or changing existing structures can be destructive and requires a migration and backup strategy.

Framework system fields are reserved and managed by the data layer:

```text
id, createdAt, createdBy, modifiedAt, modifiedBy
```

Do not declare these names as application fields.

## 5. Web Designer Development

### 5.1 Access and scope

The account must have Designer permission for the target application. The server applies an app scope to Designer operations. A user may see or customize only artifacts within the permitted application scope unless the user is a framework administrator.

The Designer stores web artifacts in the `FW_WebArtifact` table in `designer.db`. These artifacts are loaded at boot and applied to the kernel registry.

### 5.2 Simple Builder

Use Simple Builder for the first version of an application. It can create a coherent set of application metadata such as:

1. Application name, label, icon, and model settings.
2. Main table and fields.
3. Form and list fields.
4. Menu entry.
5. Initial review before applying the changes.

After creation, use the detailed Designer editors for references, indexes, actions, permissions, reports, and advanced metadata.

### 5.3 Detailed Designer workflow

The safe workflow is:

1. Select an application and artifact kind.
2. Edit the artifact in the form editor or JSON editor.
3. Validate the artifact and its cross-references.
4. Review the generated change set and schema effects.
5. Confirm the change set as a human user.
6. Apply the change set.
7. Open the generated form/list and test real CRUD behavior.

The server endpoints implementing this workflow include:

```text
GET  /api/designer/artifacts
GET  /api/designer/capabilities
GET  /api/designer/snapshot
POST /api/designer/change-sets/validate
POST /api/designer/change-sets/apply
PUT  /api/designer/artifacts/:kind/:name
DELETE /api/designer/artifacts/:kind/:name
```

### 5.4 Change sets and revisions

A change set contains a `version`, a `baseRevision`, and one or more `upsert` or `delete` operations:

```json
{
  "version": 1,
  "baseRevision": "sha256-revision-from-current-snapshot",
  "source": "designer",
  "description": "Add customer email field",
  "operations": [
    {
      "op": "upsert",
      "kind": "table",
      "name": "SALES_Customer",
      "artifact": {
        "kind": "table",
        "name": "SALES_Customer",
        "app": "sales",
        "fields": [
          { "name": "email", "type": "string", "maxLength": 200 }
        ]
      }
    }
  ]
}
```

Applying a change set requires:

- A valid preview.
- A preview that has not expired.
- The same user that created the preview.
- Human confirmation.
- The current metadata revision still matching the preview base revision.
- Additional confirmation for high-risk script changes.

If another user changes the workspace first, the apply operation returns a conflict and the change set must be validated again.

### 5.5 Export and import

Designer-managed applications and models can be exported as signed/checksummed JSON packages:

```text
GET /api/designer/packages/app/:app/export
GET /api/designer/packages/model/:app/:model/export
POST /api/designer/packages/import/preview
```

Import is previewed before application. The package scope, artifact names, app ownership, model ownership, checksum, and schema are checked. System metadata cannot be exported or imported as an application package.

## 6. JSON Metadata Structures

### 6.1 Common properties

Most artifacts support:

| Property | Meaning |
| --- | --- |
| `kind` | Artifact discriminator. |
| `name` | Stable identifier matching the framework name pattern. |
| `app` | Owning application for web artifacts. |
| `label` | Display label. |
| `model` | Optional model ownership. |
| `layer` | Optional layer override. |

Names must match the pattern `^[A-Za-z_][A-Za-z0-9_.-]*$`.

### 6.2 App manifest

```json
{
  "kind": "app",
  "name": "sales",
  "label": "Sales",
  "icon": "grid",
  "dependsOn": [],
  "models": [
    { "name": "SalesCore", "label": "Sales Core", "layer": "DEV" }
  ]
}
```

Available safe icon names are defined in `packages/core/src/metadata/types.ts`, including `app`, `grid`, `users`, `settings`, `database`, `table`, `chart`, `shield`, `wrench`, and `file`.

### 6.3 Table, field, index, enum

```json
{
  "kind": "table",
  "name": "SALES_Customer",
  "app": "sales",
  "label": "Customer",
  "titleField": "name",
  "fields": [
    {
      "name": "name",
      "type": "string",
      "label": "Customer name",
      "mandatory": true,
      "maxLength": 200
    },
    {
      "name": "status",
      "type": "enum",
      "enumName": "SALES_CustomerStatus",
      "default": 1
    },
    {
      "name": "creditLimit",
      "type": "real",
      "default": 0
    }
  ],
  "indexes": [
    { "name": "SALES_CustomerNameIdx", "fields": ["name"] },
    { "name": "SALES_CustomerUniqueNameIdx", "fields": ["name"], "unique": true }
  ]
}
```

Supported field types are:

```text
string, int, real, boolean, date, datetime, enum, reference
```

An enum is declared separately:

```json
{
  "kind": "enum",
  "name": "SALES_CustomerStatus",
  "app": "sales",
  "label": "Customer status",
  "values": [
    { "name": "Active", "value": 1, "label": "Active" },
    { "name": "Inactive", "value": 0, "label": "Inactive" }
  ]
}
```

### 6.4 Reference field

A reference stores the related record ID. The metadata controls display fields, lookup filters, delete behavior, and fields copied after selection:

```json
{
  "name": "customerId",
  "type": "reference",
  "label": "Customer",
  "mandatory": true,
  "reference": {
    "table": "SALES_Customer",
    "displayFields": ["name", "status"],
    "onDelete": "restrict",
    "filters": [
      { "field": "status", "operator": "eq", "value": 1 }
    ],
    "copyFields": [
      { "from": "creditLimit", "to": "customerCreditLimit" }
    ]
  }
}
```

`onDelete` can be `restrict`, `cascade`, or `setNull`. A mandatory reference cannot use `setNull`.

### 6.5 Form, groups, lines, and actions

```json
{
  "kind": "form",
  "name": "SALES_OrderForm",
  "app": "sales",
  "table": "SALES_Order",
  "label": "Sales order",
  "listFields": ["orderNumber", "customerId", "status", "orderDate"],
  "groups": [
    { "label": "Header", "fields": ["orderNumber", "customerId", "status"] },
    { "label": "Dates", "fields": ["orderDate"] }
  ],
  "lines": [
    {
      "table": "SALES_OrderLine",
      "refField": "orderId",
      "fields": ["productId", "quantity", "unitPrice", "lineAmount"],
      "aggregates": [
        { "fn": "sum", "field": "lineAmount", "label": "Total" }
      ]
    }
  ],
  "actions": [
    {
      "label": "Post order",
      "type": "function",
      "target": "SALES_PostOrder"
    },
    {
      "label": "Print order",
      "type": "report",
      "target": "SALES_OrderReport"
    }
  ]
}
```

The legacy `action` property is still supported, but new metadata should prefer `type` and `target`.

### 6.6 Record picker action

```json
{
  "label": "Allocate stock",
  "type": "picker",
  "target": "SALES_AllocateStock",
  "picker": {
    "table": "SALES_Stock",
    "columns": ["productId", "warehouseId", "availableQuantity"],
    "searchFields": ["productId", "warehouseId"],
    "multiple": true,
    "filters": [
      {
        "field": "warehouseId",
        "operator": "eq",
        "value": { "source": "record", "field": "warehouseId" }
      }
    ],
    "allocation": {
      "availableField": "availableQuantity",
      "quantityLabel": "Quantity to allocate"
    }
  }
}
```

Picker filter values may be constants or references to a field on the current record or current line.

### 6.7 Menu

```json
{
  "kind": "menu",
  "name": "SALES_MainMenu",
  "app": "sales",
  "label": "Sales",
  "items": [
    {
      "label": "Customers",
      "icon": "users",
      "target": { "type": "form", "name": "SALES_CustomerForm" }
    },
    {
      "label": "Orders",
      "icon": "table",
      "target": { "type": "form", "name": "SALES_OrderForm" }
    },
    {
      "label": "Reports",
      "items": [
        {
          "label": "Order report",
          "target": { "type": "report", "name": "SALES_OrderReport" }
        }
      ]
    }
  ]
}
```

### 6.8 Security metadata

```json
{
  "kind": "privilege",
  "name": "SALES_OrderPrivilege",
  "app": "sales",
  "label": "Order access",
  "tablePermissions": [
    {
      "table": "SALES_Order",
      "read": true,
      "create": true,
      "update": true,
      "delete": false
    }
  ],
  "forms": ["SALES_OrderForm"],
  "functions": ["SALES_PostOrder"],
  "reports": ["SALES_OrderReport"]
}
```

```json
{
  "kind": "duty",
  "name": "SALES_OrderClerkDuty",
  "app": "sales",
  "privileges": ["SALES_OrderPrivilege"]
}
```

```json
{
  "kind": "role",
  "name": "SALES_OrderClerkRole",
  "app": "sales",
  "duties": ["SALES_OrderClerkDuty"]
}
```

### 6.9 Extensions

Extensions add to existing metadata and should be independently removable:

```json
{
  "kind": "tableExtension",
  "name": "SALES_CustomerLocalization",
  "app": "sales",
  "table": "SALES_Customer",
  "layer": "LOC",
  "fields": [
    { "name": "localName", "type": "string", "label": "Local name" }
  ],
  "indexes": [
    { "name": "SALES_CustomerLocalNameIdx", "fields": ["localName"] }
  ]
}
```

Other extension shapes use the target property appropriate to the kind: `form`, `menu`, `enum`, `privilege`, `duty`, `role`, or `script`.

### 6.10 Report

Reports use bands and positioned elements:

```json
{
  "kind": "report",
  "name": "SALES_OrderReport",
  "app": "sales",
  "label": "Sales order report",
  "dataSource": "SALES_Order",
  "page": {
    "size": "A4",
    "orientation": "portrait",
    "margins": [36, 36, 36, 36]
  },
  "bands": [
    {
      "kind": "header",
      "height": 40,
      "elements": [
        {
          "id": "title",
          "type": "text",
          "x": 36,
          "y": 10,
          "width": 300,
          "height": 20,
          "text": "Sales Order",
          "style": { "fontSize": 16, "bold": true }
        }
      ]
    },
    {
      "kind": "detail",
      "height": 24,
      "elements": [
        {
          "id": "orderNumber",
          "type": "field",
          "x": 36,
          "y": 4,
          "width": 180,
          "height": 16,
          "field": "orderNumber"
        }
      ]
    }
  ]
}
```

## 7. Complete Application Example

The following artifacts form a small but coherent application:

```text
sales
├── App manifest
├── SALES_CustomerStatus enum
├── SALES_Customer table
├── SALES_Order table
├── SALES_OrderLine table
├── SALES_OrderForm form
├── SALES_MainMenu menu
├── SALES_OrderPrivilege privilege
├── SALES_OrderClerkDuty duty
└── SALES_OrderClerkRole role
```

The dependency direction should be:

```text
enum → tables → forms/reports → menus → privileges → duties → roles
```

Create referenced tables and enums before forms or actions that use them. The registry validates cross-references during application.

## 8. Business Logic Development

Use the smallest mechanism that matches the behavior:

| Requirement | Recommended mechanism |
| --- | --- |
| Set a default on a new record | `initValue` hook or field `default` |
| Validate every write | `validateWrite` hook or `onValidating` event |
| Validate deletion | `validateDelete` hook or `onDeleting` event |
| React to a lifecycle stage | Data event |
| Run an explicit user action | Named `function` artifact |
| Add several related handlers | `script` artifact |

Keep business rules deterministic and server-side. The client may hide a button, but authorization must still be enforced by the server data policy and action path.

## 9. Function and Script Development

### 9.1 Function metadata

A Function artifact contains a body compiled by the kernel:

```json
{
  "kind": "function",
  "name": "SALES_PostOrder",
  "app": "sales",
  "label": "Post order",
  "code": "const orderId = Number(args.orderId);\nconst order = ctx.find('SALES_Order', orderId);\nif (!order) throw new Error('Order not found');\nif (order.f.status !== 1) throw new Error('Only open orders can be posted');\norder.set('status', 2).update();\nreturn { ok: true, orderId };"
}
```

The body is invoked with:

```ts
(ctx, args, kernel)
```

The function receives:

- `ctx`: the authenticated `DataContext` for the request.
- `args`: the JSON request body or action arguments.
- `kernel`: the active kernel, including registry, actions, events, hooks, and runtime services.

The function result becomes the API response. If it returns `undefined`, the server returns `{ ok: true }`.

### 9.2 Calling a function from an action button

Use a Form Action:

```json
{
  "label": "Post order",
  "type": "function",
  "target": "SALES_PostOrder"
}
```

The named action API is:

```text
POST /api/action/:name
```

The server resolves the name from `kernel.actions`, creates the authenticated context, and runs the handler inside `ctx.tts()`. Unknown action names return HTTP 404.

### 9.3 Transactional function

The action route already wraps the handler in a transaction. For internal code that performs a group of writes, use `ctx.tts()` explicitly when the caller does not already provide a transaction:

```js
return ctx.tts(() => {
  const header = ctx.newRecord('SALES_Order');
  header.setMany({
    orderNumber: String(args.orderNumber),
    status: 1,
    customerId: Number(args.customerId)
  }).insert();

  for (const line of args.lines ?? []) {
    ctx.newRecord('SALES_OrderLine').setMany({
      orderId: header.id,
      productId: Number(line.productId),
      quantity: Number(line.quantity),
      unitPrice: Number(line.unitPrice)
    }).insert();
  }

  return { ok: true, orderId: header.id };
});
```

Transactions use SQLite `BEGIN`/`COMMIT`. Nested calls use savepoints. Any thrown error rolls back the current transaction or savepoint.

### 9.4 Script metadata

A Script artifact is a registration script. It is compiled as:

```ts
new Function('kernel', 'ValidationError', 'DataEventCancelled', code)
```

Example:

```json
{
  "kind": "script",
  "name": "SALES_OrderRules",
  "app": "sales",
  "label": "Order rules",
  "code": "kernel.hooks.register('SALES_Order', {\n  initValue(record) {\n    if (record.get('status') === null) record.set('status', 1);\n  },\n  validateWrite(record) {\n    if (Number(record.get('totalAmount') ?? 0) < 0) {\n      throw new ValidationError('Order total cannot be negative');\n    }\n  }\n});\n\nkernel.events.on('SALES_Order', 'onInserting', (event) => {\n  if (!event.record.get('orderNumber')) event.cancel('Order number is required');\n});"
}
```

Script code should register hooks, events, or actions. Keep it short and testable. Native integrations, external services, and complex reusable logic should normally be implemented as reviewed TypeScript rather than embedded dynamic code.

### 9.5 Registering an action in a script

```js
kernel.actions.set('SALES_RecalculateOrder', (ctx, args) => {
  const order = ctx.find('SALES_Order', Number(args.orderId));
  if (!order) throw new Error('Order not found');

  const total = ctx
    .select('SALES_OrderLine')
    .whereEq({ orderId: order.id })
    .toArray()
    .reduce((sum, line) => sum + Number(line.f.lineAmount ?? 0), 0);

  order.set('totalAmount', total).update();
  return { ok: true, totalAmount: total };
});
```

Function artifacts are registered after web scripts. A Function artifact with the same action name can therefore replace a script-registered action according to the kernel registration behavior. Avoid duplicate names unless intentional and documented.

## 10. Hooks and Data Events

### 10.1 Hooks

Hooks are table-level lifecycle methods:

```ts
interface TableHooks {
  initValue?(record: Record, ctx: DataContext): void;
  validateWrite?(record: Record, ctx: DataContext): boolean | void;
  validateDelete?(record: Record, ctx: DataContext): boolean | void;
}
```

Returning `false` from a validation hook blocks the operation. Throwing a `ValidationError` provides a more specific message.

Example:

```js
kernel.hooks.register('SALES_Customer', {
  initValue(record) {
    if (record.get('status') === null) record.set('status', 1);
  },
  validateWrite(record) {
    const name = String(record.get('name') ?? '').trim();
    if (!name) throw new ValidationError('Customer name is required');
    if (name.length > 200) throw new ValidationError('Customer name is too long');
  },
  validateDelete(record, ctx) {
    const hasOrders = ctx.select('SALES_Order').whereEq({ customerId: record.id }).count() > 0;
    return !hasOrders;
  }
});
```

### 10.2 Events

The supported events are:

| Event | Operation | Can cancel? |
| --- | --- | --- |
| `onValidating` | Common write validation stage | Yes |
| `onInserting` | Immediately before insert | Yes |
| `onInserted` | Immediately after insert | No |
| `onUpdating` | Immediately before update | Yes |
| `onUpdated` | Immediately after update | No |
| `onDeleting` | Immediately before delete | Yes |
| `onDeleted` | Immediately after delete | No |

Event handlers receive:

```ts
{
  table: string;
  event: DataEventType;
  record: Record;
  ctx: DataContext;
  cancel(reason: string): never;
}
```

Example:

```js
kernel.events.on('SALES_Order', 'onValidating', (event) => {
  const quantity = Number(event.record.get('totalAmount') ?? 0);
  if (quantity < 0) event.cancel('Order total cannot be negative');
});

kernel.events.on('SALES_Order', 'onInserted', (event) => {
  // Notification or audit behavior can be placed here.
  // It cannot cancel the already-completed insert.
  event.ctx.newRecord('SALES_OrderAudit').setMany({
    orderId: event.record.id,
    message: 'Order created'
  }).insert();
});
```

Calling `cancel()` during a post-event raises an error because post-events cannot cancel an operation that has already completed. Cancellation is represented by `DataEventCancelled`.

### 10.3 Write lifecycle

For an insert, the effective sequence is:

```text
permission check
→ field validation
→ onValidating
→ validateWrite hooks
→ onInserting
→ system audit fields
→ SQLite INSERT
→ onInserted
```

For an update, the sequence is analogous, using `onUpdating` and `onUpdated`.

Delete processing also handles reference relationships. `restrict` blocks deletion, `setNull` clears child references, and `cascade` deletes children. Child writes still pass through their normal policy and lifecycle behavior.

## 11. Execution, Transactions, and Security

### 11.1 Execution order

The kernel orders web scripts using layer and name ordering. Metadata registration occurs before function artifacts are compiled. Function artifacts are sorted by layer and name, then registered in `kernel.actions`.

Do not rely on accidental registration order. Use unique names and explicit dependencies where possible.

### 11.2 Authorization

`DataContext` receives a session and security policy. Every read and write path checks the policy before executing SQL. A UI permission or hidden button is not a security boundary.

When writing a new server action:

- Use the authenticated context supplied by the server.
- Read and write through `ctx`.
- Do not create an unrestricted context to bypass user permissions.
- Validate that the user may operate on the target record.
- Test the action with an authorized and unauthorized user.

### 11.3 Error handling

Use validation errors for user-correctable data problems:

```js
if (!record.get('customerId')) {
  throw new ValidationError('Customer is required');
}
```

Use ordinary errors for unexpected or infrastructure failures. Do not expose secrets, SQL details, or internal stack traces in user-facing messages.

### 11.4 Dynamic code safety

Scripts and functions are dynamic server-side code compiled with `new Function`. Treat them as trusted administrative code. Restrict Designer access, review scripts, back up databases before high-risk changes, and prefer normal TypeScript for code requiring stronger review, typing, or external integration.

## 12. CLI and File-based Apps

The CLI scaffolds an application directory with metadata folders and a TypeScript logic file:

```text
apps/<app-name>/
├── app.json
├── package.json
├── tsconfig.json
├── src/
│   └── logic.ts
├── metadata/
│   ├── tables/
│   ├── enums/
│   ├── forms/
│   ├── menus/
│   ├── privileges/
│   ├── duties/
│   ├── roles/
│   └── menuExtensions/
└── test/
```

The scaffolding implementation is in `packages/cli/src/scaffold/`. Prefer the CLI for repetitive, file-based metadata creation:

```sh
pnpm emu --help
pnpm emu add-app <name>
pnpm emu add-object <type> <name>
pnpm emu add-extension <type> <name>
```

Use Web Designer for runtime/customizer-owned metadata and file-based metadata for source-controlled application definitions. Do not overwrite framework files to customize an existing application; use extensions or a documented higher layer.

## 13. Testing and Debugging

### 13.1 Test categories

| Area | What to test |
| --- | --- |
| Metadata | Schema validity, references, duplicates, layers, extensions, and destructive changes. |
| Core data API | Create, read, update, delete, validation, hooks, events, references, and transactions. |
| Security | Every CRUD operation and named action with allowed and denied users. |
| Server | Fastify routes, authentication, Designer workflow, packages, reports, and import/export. |
| Client | Navigation, generated forms, Designer editors, and client utilities. |
| CLI | Scaffold output, paths, JSON shape, and duplicate handling. |
| Deployment | Windows launcher, Docker configuration, update, backup, restore, and recovery. |

### 13.2 Minimum business logic tests

For every new hook, event, or function, test:

1. Valid input succeeds.
2. Invalid input is rejected with a useful error.
3. A transaction rolls back all related writes after failure.
4. Unauthorized access is rejected.
5. Updates do not accidentally bypass validation.
6. Delete behavior matches the reference `onDelete` setting.
7. Post-events are not expected to cancel operations.

Existing examples can be found in:

- `packages/core/test/dataApi.test.ts`
- `packages/core/test/webArtifacts.test.ts`
- `packages/server/test/designer.test.ts`
- `packages/server/test/appAccess.test.ts`
- `packages/server/test/metadataPackage.test.ts`

### 13.3 Debugging checklist

When metadata does not appear:

1. Check the artifact `kind` and `name`.
2. Validate the JSON shape.
3. Check app and model scope.
4. Check references to tables, fields, forms, enums, and actions.
5. Check the layer and whether a higher-layer artifact overrides it.
6. Inspect the Designer/API response for diagnostics.
7. Reload the Web Designer runtime if required.
8. Verify that the generated form/list uses the expected artifact.

When a Function fails:

1. Confirm the action name matches exactly.
2. Confirm the request body contains the expected `args` fields.
3. Confirm the current user has table permissions.
4. Check that `ctx.find()` returned a record.
5. Check field types before calling `set()`.
6. Check whether an exception triggered a rollback.
7. Check for another function or script registering the same action name.

## 14. Feature Development Checklist

### Design

- [ ] Define stable artifact names and user-facing labels.
- [ ] Decide whether the feature belongs in base metadata or an extension.
- [ ] Identify tables, references, enums, forms, menus, reports, and permissions.
- [ ] Choose the correct layer.

### Implementation

- [ ] Add or edit metadata.
- [ ] Validate JSON and cross-references.
- [ ] Use `DataContext` for all data access.
- [ ] Put validation in hooks/events, not only in the client.
- [ ] Wrap related writes in a transaction when needed.
- [ ] Enforce authorization on the server.
- [ ] Use unique function and script names.

### Verification

- [ ] Test create, update, delete, and validation failure paths.
- [ ] Test rollback behavior.
- [ ] Test permissions with multiple roles.
- [ ] Test generated forms, lists, actions, and menus.
- [ ] Test import/export if the artifact is packageable.
- [ ] Run `pnpm typecheck`, `pnpm test`, and `pnpm build`.
- [ ] Review schema effects and take a backup before destructive changes.

## 15. Appendix

### 15.1 Data field types

| Type | Stored/expected value |
| --- | --- |
| `string` | String. |
| `int` | Number. |
| `real` | Number, including decimals. |
| `boolean` | Boolean in application code; SQLite stores it as `0` or `1`. |
| `date` | String date value. |
| `datetime` | String datetime value. |
| `enum` | Numeric enum value linked to an enum artifact. |
| `reference` | Numeric ID of a record in another table. |

### 15.2 Core data API examples

```ts
const ctx = kernel.context({ user: 'alice' }, policy);

const customers = ctx
  .select('SALES_Customer')
  .whereEq({ status: 1 })
  .orderBy('name', 'asc')
  .toArray();

const customer = ctx.find('SALES_Customer', 10);

const newCustomer = ctx
  .newRecord('SALES_Customer')
  .setMany({ name: 'Example Customer', status: 1 });
newCustomer.insert();

ctx.tts(() => {
  newCustomer.set('creditLimit', 10000).update();
});
```

### 15.3 Important API routes

```text
GET/POST/PATCH/DELETE /api/data/:table
POST                 /api/action/:name
GET                  /api/designer/artifacts
POST                 /api/designer/change-sets/validate
POST                 /api/designer/change-sets/apply
GET                  /api/designer/packages/app/:app/export
POST                 /api/designer/packages/import/preview
```

### 15.4 Primary source files

- [Metadata types](https://github.com/emu479p01/emu-framework/blob/master/packages/core/src/metadata/types.ts)
- [Metadata schema](https://github.com/emu479p01/emu-framework/blob/master/packages/core/src/metadata/schema.ts)
- [Data context](https://github.com/emu479p01/emu-framework/blob/master/packages/core/src/data/context.ts)
- [Data events](https://github.com/emu479p01/emu-framework/blob/master/packages/core/src/data/events.ts)
- [Data hooks](https://github.com/emu479p01/emu-framework/blob/master/packages/core/src/data/hooks.ts)
- [Kernel](https://github.com/emu479p01/emu-framework/blob/master/packages/core/src/kernel.ts)
- [Designer server routes](https://github.com/emu479p01/emu-framework/blob/master/packages/server/src/designer.ts)
- [Server action route](https://github.com/emu479p01/emu-framework/blob/master/packages/server/src/server.ts)
- [Designer client views](https://github.com/emu479p01/emu-framework/tree/master/packages/client/src/views/designer)
- [CLI scaffolding](https://github.com/emu479p01/emu-framework/tree/master/packages/cli/src/scaffold)

### 15.5 Existing documentation

- [Documentation index](../README.md)
- [Web Designer user guide](../user/web-designer.md)
- [Developer architecture guide](architecture.md)
- [Metadata development guide](metadata.md)
- [Business logic guide](business-logic.md)
- [Testing guide](testing.md)
