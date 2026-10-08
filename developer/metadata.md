# Work with metadata

## Purpose

Define applications, data structures, UI, security, reports, and server behavior as validated metadata artifacts.

## Audience

Application developers and Web Designer customizers.

## Prerequisites

A configured development environment or Designer permission for the target app.

## Artifact lifecycle

```mermaid
flowchart LR
    A[Artifact JSON] --> B[Shape validation]
    B --> C[Cross-reference validation]
    C --> D[Layer and dependency resolution]
    D --> E[Registry]
    E --> F[UI and API]
    E --> G[Database synchronization]
```

## Concepts

Metadata describes Apps, Models, tables, fields, enums, Forms, menus, Privileges, Duties, Roles, Reports, Views, Charts, Scripts, Functions, Translations, and Data Entities. `name` is a stable identifier; `label` is user-facing text. Supported base kinds include `app`, `enum`, `table`, `form`, `menu`, `privilege`, `duty`, `role`, `script`, `function`, `report`, `translation`, `dataEntity`, `view`, and `chart`. [Translations](localization.md) provide per-locale labels, and [Data Entities](data-entities.md) describe documents for spreadsheet exchange and archiving.

Metadata is resolved through ordered layers: `SYS < ISV < LOC < DEV < CUS`. Base artifacts at a higher layer override lower-layer artifacts with the same logical identity; Extensions accumulate into their target. See [Work with metadata layers](layers.md) for ownership and precedence rules.

Applications contain named Models, and Models provide the organizational and ownership context for their metadata. Review [Understand Apps, Models, and Layers](app-model-layer.md) before deciding where an artifact belongs.

Prefer Web Designer or a reviewed AI REST proposal over repetitive handwritten payloads. Validate relationships, enum values, menu targets, action targets, and security Artifacts together.

## Schema and database rules

Schema synchronization is additive: adding tables, fields, and indexes is supported. Removing or changing existing structures needs an explicit migration and backup plan. Do not declare framework system fields as application fields. The reserved names are `id`, `createdAt`, `createdBy`, `modifiedAt`, `modifiedBy`, and the virtual aliases `sys_createdBy`, `sys_createdAt`, `sys_modifiedBy`, and `sys_modifiedAt`, compared case-insensitively. If a stored table already has a physical column that collides with an alias, schema synchronization stops before it changes anything. See [Understand the record lifecycle](record-lifecycle.md). Enum fields and fields with `readOnly: true` must not also set `mandatory: true`. String fields can set `multiline` and `encrypted`; an encrypted field cannot define a default.

## Procedure

1. Choose a stable name, owning App, existing Model, and matching Layer.
2. Define referenced enums and tables before Forms, Views, Charts, Reports, menus, and actions.
3. Validate the artifact shape and cross-references.
4. Review layer, dependencies, and schema effects.
5. Run the application and exercise generated list and form pages.

## Metadata API

These session-cookie endpoints are the internal interface used by Web Designer. They require Designer/Customize permission for the target App and are not the external AI integration contract. External AI clients must use the scoped [AI REST proposal API](ai-rest-api.md), which never applies changes directly.

For the complete request flow, required properties, status codes, and ChangeSet confirmation contract, read [Create metadata Artifacts through the API](artifact-api.md). Use [Artifact kind reference](artifact-types.md) for every supported `kind` and [Nested metadata structures](artifact-components.md) for Fields, actions, Line grids, menu items, Report bands, View expressions, and Extension overrides.

`GET /api/designer/artifacts` supports `app`, `model`, `kind`, `cursor`, and `limit` query parameters, returns `nextCursor` and `total`, and emits an ETag based on the metadata revision. Request `includeCatalog=false` when the caller needs only the stored Artifact page; use `/api/designer/catalog` for the separately cached effective catalog.

### Create an object

```http
POST /api/designer/artifacts
Content-Type: application/json
```

The body is a complete artifact and must include `kind`, `name`, `app`, `model`, and the selected Model's `layer` for business objects. This endpoint is create-only and returns `409` when the name already exists.

```json
{
  "kind": "enum",
  "name": "SALES_OrderStatus",
  "app": "sales",
  "model": "Customizations",
  "layer": "CUS",
  "values": [
    { "name": "Open", "value": 0 },
    { "name": "Confirmed", "value": 1 }
  ]
}
```

A successful response returns status `201`.

### Create or update idempotently

```http
PUT /api/designer/artifacts/{kind}/{name}
```

Web Designer uses this operation when it needs to save the same object idempotently. The `kind` and `name` in the URL are authoritative.

### Create multiple objects atomically

Use the change-set workflow:

1. `GET /api/designer/snapshot` to obtain the current `revision`.
2. `POST /api/designer/change-sets/validate` with the intended operations.
3. Review the diff, warnings, and any high-risk flags.
4. `POST /api/designer/change-sets/apply` with the returned `previewId` and human confirmation.

A change set is the right tool when creating a Table + Form + Menu + Security graph inside an existing App/Model, since it never leaves the system in a half-applied state. Create the App, then add its Model, before validating that graph.

### Supported object kinds

`app`, `table`, `enum`, `form`, `menu`, `script`, `function`, `report`, `translation`, `dataEntity`, `view`, `chart`, `privilege`, `duty`, `role`, `tableExtension`, `enumExtension`, `formExtension`, `menuExtension`, `privilegeExtension`, `dutyExtension`, `roleExtension`, `scriptExtension`, `viewExtension`, `chartExtension`, `functionExtension`, `dataEntityExtension`

An App is created with `models: []`. Add a Model through Designer before creating other Artifacts. System metadata appears only to a System Administrator in the **Framework — Read-only** scope and is rejected by every mutation and packaging endpoint.

Every API channel validates schema, naming, app/model/layer, dependencies, cross-references, and permissions before saving.

## Metadata delivered to the client

`GET /api/metadata` returns the metadata the signed-in user may use, already localized for the user's locale. It is not the Designer contract. Notable properties are:

- `tables` includes the system fields (`id`, the audit columns, and the audit aliases) in each Table's `fields`.
- `forms` contain only the actions the user is authorized to run. A form or line action whose Function, Report, Picker table, or Privilege is not granted is omitted, and a direct call to its endpoint returns `403`. Do not depend on a disabled button; the server filters the list.
- `functionInputs` lists `name`, `label`, and `imageInput` for each Function the user may run. Function code is never sent.
- `dataEntities` lists the entities whose root and line tables the user can read.
- `locale`, `availableLocales`, `uiMessages`, and `apps[].defaultLocale` / `apps[].availableLocales` describe localization; see [Localize metadata with Translations](localization.md).
- `branding.title` carries the product name set by the `EMU_APP_TITLE` environment variable (default `EmuFramework`), and `licenseNotices` reports expiring or blocking ISV licenses.

## Related topics

[Artifact API](artifact-api.md) · [Artifact kinds](artifact-types.md) · [Nested structures](artifact-components.md) · [Localization](localization.md) · [Data Entities](data-entities.md) · [Record lifecycle](record-lifecycle.md) · [AI REST API](ai-rest-api.md) · [Security](security.md) · [Web Designer](../user/web-designer.md)
