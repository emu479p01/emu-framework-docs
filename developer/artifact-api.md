# Create metadata Artifacts through the API

## Purpose

Create, update, validate, and apply EmuFramework v1.4.0 metadata with the exact wire format accepted by the server.

## Choose the API

EmuFramework exposes two authoring paths with different security properties:

| Path | Authentication | Can apply metadata? | Intended caller |
| --- | --- | ---: | --- |
| `/api/designer/*` | Signed-in session cookie with Customize permission | Yes, after validation and human confirmation | Web Designer and trusted same-origin tools |
| `/api/v1/ai/*` | Dedicated Bearer token scoped to existing Apps | No; it can only validate and propose | AI integrations |

The Designer API is not a general anonymous service. A normal user needs `canCustomize=true` for every affected App; `FW_SystemAdminRole` can customize all business Apps. Framework metadata (`system` and names beginning `FW_`) is read-only.

## Read the live contract first

Use the server's schemas instead of copying an old payload:

```text
GET /api/designer/capabilities
GET /api/v1/ai/capabilities
GET /api/v1/ai/schemas/artifact
GET /api/v1/ai/schemas/change-set
```

The artifact schema rejects unknown properties because every object uses `additionalProperties: false`. A misspelled field is an error, not an ignored setting.

## Required placement and identity

Every non-App Artifact sent to a creation API must include:

```json
{
  "kind": "table",
  "name": "SALES_Order",
  "app": "sales",
  "model": "Core",
  "layer": "ISV",
  "fields": []
}
```

`kind`, `name`, `app`, and `model` are required in practice. `layer` is optional in the JSON Schema, but including it is recommended for readable exports. The effective Layer always comes from the selected Model; changing the payload's `layer` alone does not move an Artifact to another Layer.

Identity rules:

- `name` matches `^[A-Za-z_][A-Za-z0-9_.-]*$`.
- Business Artifact names start with the upper-case prefix derived from the first App segment. App `sales` uses `SALES_`; App `erp.credit` uses `ERP_`.
- Names are global identities. An existing Artifact cannot change `kind` or move to another App.
- `model` must exist in the App manifest. A zero-Model App cannot contain other Artifacts.
- The server uses the Model's Layer from `SYS < ISV < LOC < DEV < CUS`.
- `label` is generally optional and user-facing; `name` remains the stable identifier.

## Create an App and Model

An App is the only Artifact without `app`, `model`, or `layer`:

```http
POST /api/designer/artifacts
Content-Type: application/json
```

```json
{
  "kind": "app",
  "name": "sales",
  "label": "Sales",
  "icon": "app",
  "dependsOn": [],
  "models": []
}
```

`models` may be omitted; the API normalizes it to `[]`. Add a Model before creating other Artifacts:

```http
PUT /api/designer/artifacts/model/sales/Core
Content-Type: application/json
```

```json
{
  "label": "Core application",
  "layer": "ISV"
}
```

`layer` is required and must be `SYS`, `ISV`, `LOC`, `DEV`, or `CUS`. A Model is stored inside the App manifest; `model` is not an Artifact `kind`.

An `ISV` Model can also carry `"license": { "vendor": "seller-id" }` to require an offline license. Once a requirement exists, a later request that changes the vendor or moves the Model off `ISV` returns `422` (`Cannot change an installed license requirement`). An App update that drops or alters the requirement returns `422` (`Cannot remove or change license requirement for '<model>'`), and deleting the Model through this API is refused. See [Deploy models and license ISV add-ons](model-deployment.md).

To set the App's language, send `defaultLocale` (a BCP-47 tag) on the `app` Artifact. An invalid tag returns a `422` diagnostic at `/defaultLocale`.

AI tokens can target only existing non-system Apps, so the first App must be created through Web Designer or the authenticated Designer API.

## Create one Artifact

Use create-only POST when overwriting an existing name must be impossible:

```http
POST /api/designer/artifacts
Content-Type: application/json
Cookie: nf_session=<session>
```

The body is one complete Artifact. Success returns HTTP `201`. An existing name returns `409`; schema or registry validation normally returns `422`.

## Create or update one Artifact

Use idempotent PUT:

```http
PUT /api/designer/artifacts/{kind}/{name}
Content-Type: application/json
Cookie: nf_session=<session>
```

The URL's `kind` and `name` are authoritative and replace those values in the body. Send the complete desired Artifact, not a JSON Merge Patch. The entire candidate workspace is revalidated before persistence.

## Kinds added since v0.5.0.0

The Designer API accepts `translation`, `dataEntity`, and `dataEntityExtension` in addition to the earlier kinds, with the same placement rules. Check them against `GET /api/designer/capabilities`, which returns the live Artifact and ChangeSet schemas. Schema diagnostics also report these rules:

| Rule | Diagnostic |
| --- | --- |
| `locale` or `defaultLocale` is not a valid tag | `invalid_locale` |
| A Model other than `ISV` declares `license` | `license_layer` |
| An enum field is mandatory | `enum_optional` |
| A read-only field is mandatory | `readonly_optional` |
| `multiline` or `encrypted` is set on a non-string field | `field_rules` |
| An encrypted field defines a `default` | `field_rules` |

Related Designer endpoints are `GET /api/designer/translations/diagnostics` for duplicate and stale Translation keys, and `POST /api/designer/reports/validate` with `{ "artifact": <report> }`, which returns `valid`, layout `diagnostics` with `severity` `error` or `warning`, and a `summary` (see [Design paginated Reports](reports.md)). `POST /api/designer/packages/models/:app/export` builds a selected-model package and `POST /api/designer/packages/import/preview` previews any package; both are described in [Deploy models and license ISV add-ons](model-deployment.md).

## Apply several dependent Artifacts atomically

Use a ChangeSet for graphs such as Enum → Table → Form → Menu → Privilege. First read the workspace revision:

```http
GET /api/designer/snapshot?app=sales
```

Then validate a version 1 ChangeSet:

```http
POST /api/designer/change-sets/validate
Content-Type: application/json
```

```json
{
  "version": 1,
  "baseRevision": "<revision from snapshot>",
  "source": "designer",
  "description": "Create sales order metadata",
  "operations": [
    {
      "op": "upsert",
      "kind": "enum",
      "name": "SALES_OrderStatus",
      "artifact": {
        "kind": "enum",
        "name": "SALES_OrderStatus",
        "app": "sales",
        "model": "Core",
        "layer": "ISV",
        "values": [{ "name": "Open", "value": 0 }]
      }
    }
  ]
}
```

Rules:

- `version` is exactly `1`.
- `baseRevision` is required and must still equal the current workspace revision.
- `operations` contains at least one `upsert` or `delete`.
- An upsert operation's `kind` and `name` must equal its nested Artifact identity.
- Set `source: "designer"` when an authenticated Designer ChangeSet contains executable Scripts or Functions.
- Deleting an App or Table is high-risk and preserves its physical business tables as orphans.

A valid response includes `previewId`, `expiresAt`, `diff`, `schemaEffects`, warnings, and the next revision. The preview expires after 10 minutes. Apply it with explicit confirmation:

```http
POST /api/designer/change-sets/apply
Content-Type: application/json
```

```json
{
  "previewId": "<previewId>",
  "confirmation": true,
  "confirmHighRisk": false
}
```

Set `confirmHighRisk: true` when the preview contains an executable Artifact or another high-risk diff. A preview belongs to the validating user. If the workspace changes before apply, the server returns `409` and requires a new validation.

Apply revalidates the candidate workspace and then applies it as one unit. When applying the registry or saving the Artifacts fails, runtime registrations (hooks, events, actions) and database changes made during the apply are restored, so a failed apply does not leave a partial runtime.

## Recommended dependency order

Within one ChangeSet, include every new dependency. The registry sorts base Artifacts before Extensions, but the clearest authoring order is:

```text
App → Model → Enum/Table → View → Chart → Form/Report/Function/Script
    → Menu → Privilege → Duty → Role → Extension
```

App and Model usually exist before the ChangeSet because Model creation has its own endpoint.

## Delete behavior

```http
DELETE /api/designer/artifacts/{kind}/{name}
```

Deleting metadata does not immediately drop the physical SQLite table. Deleting an App cascades its stored metadata and reports owned tables in `orphanedTables`. Only a Framework Administrator can purge an orphan, and the request must repeat the exact table name as confirmation.

## Status codes and diagnostics

| Status | Meaning |
| ---: | --- |
| `201` | Artifact or proposal created. |
| `400` | Missing input, unsupported kind, or missing confirmation. |
| `401` | Missing/invalid session or AI token. |
| `403` | No Customize/App/token scope, or Framework metadata is read-only. |
| `404` | A requested route resource, such as an App or stored Artifact, does not exist. |
| `409` | Duplicate create, stale workspace, invalid move, or concurrent job. |
| `410` | Preview expired. |
| `422` | JSON Schema, placement, missing target, or cross-reference/registry validation failed. |

Schema failures include a JSON path in `diagnostics`. Cross-reference failures appear in `registryErrors` or `error`. Fix all errors before retrying; field order does not make an invalid dependency valid.

## Reference

- [Artifact kinds](artifact-types.md) lists required and optional properties for every supported `kind`, including `translation`, `dataEntity`, and `dataEntityExtension`.
- [Nested metadata structures](artifact-components.md) defines fields, references, actions, menus, report bands, View expressions, and overrides.
- [AI REST proposal API](ai-rest-api.md) documents the non-applying Bearer-token workflow.

## Related topics

[Metadata](metadata.md) · [Artifact kinds](artifact-types.md) · [Nested structures](artifact-components.md) · [Extensions](extensions.md) · [Security](security.md)
