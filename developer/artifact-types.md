# Artifact kind reference

## How to read this reference

Every non-App example assumes App `sales` and the named Model already exist. Referenced targets must exist in the same candidate workspace or in a declared dependency. The examples show the smallest useful shape; see [Nested metadata structures](artifact-components.md) for every child property.

Common non-App properties are:

| Property | API requirement | Notes |
| --- | --- | --- |
| `kind` | Required | One of the kinds below. |
| `name` | Required | Stable global identity and App prefix. |
| `app` | Required | Existing business App. |
| `model` | Required | Existing Model inside that App. |
| `layer` | Recommended | Effective value is the Model Layer. |
| `label` | Optional on supported kinds | User-facing text. Not every Extension schema accepts it. |

## Supported kinds at a glance

| Kind | Kind-specific required properties | Main optional properties |
| --- | --- | --- |
| `app` | `name` | `label`, `icon`, `dependsOn`, `models` |
| `table` | `fields` | `label`, `titleField`, `indexes` |
| `enum` | `values` | `label` |
| `form` | `table` | `label`, `actions`, `listFields`, `filterFields`, `groups`, `charts`, `lines` |
| `menu` | `items` | `label` |
| `privilege` | none beyond common | `tablePermissions`, `forms`, `functions`, `reports`, `views` |
| `duty` | `privileges` | `label` |
| `role` | none beyond common | `label`, `duties`, `privileges` |
| `script` | `code` | `label` |
| `function` | `code` | `label`, `executionMode`, `privileges` |
| `report` | `dataSource`, `bands` | `label`, `defaultFont`, `privileges`, `page`, `lineSources`, `parameters` |
| `view` | `source`, non-empty `columns` | `label`, `joins`, `parameters`, `filters`, `groupBy`, `orderBy` |
| `chart` | `type`, `view`, non-empty `measures` | `label`, `dimension`, `legend`, `stacked` |
| `tableExtension` | `table` | `fields`, `indexes`, `fieldOverrides` |
| `formExtension` | `form` | `listFields`, `filterFields`, `groups`, `charts`, `actions`, `lines`, `lineOverrides`, `elementOverrides` |
| `menuExtension` | `menu` | `items`, `insertions`, `itemOverrides` |
| `enumExtension` | `enum`, `values` | `valueOverrides` |
| `privilegeExtension` | `privilege` | permission collections |
| `dutyExtension` | `duty` | `privileges` |
| `roleExtension` | `role` | `duties`, `privileges` |
| `scriptExtension` | `script`, `code` | none beyond common placement |
| `functionExtension` | `function`, `code` | none beyond common placement |
| `viewExtension` | `view` | `joins`, `columns`, `filters`, `orderBy`, `columnOverrides` |
| `chartExtension` | `chart` | `measures`, `label`, `legend`, `stacked`, `measureOverrides` |

There is no `reportExtension` kind in v0.5.0.0.

## App

`app` requires only `kind` and `name`. `models` may be empty. `icon` must come from the safe icon catalog. Every `dependsOn` App must be loaded when cross-App Artifacts are resolved.

```json
{
  "kind": "app",
  "name": "sales",
  "label": "Sales",
  "icon": "app",
  "dependsOn": [],
  "models": [
    { "name": "Core", "label": "Core", "layer": "ISV" },
    { "name": "Customizations", "label": "Customizations", "layer": "CUS" }
  ]
}
```

Use the dedicated Model endpoint when adding or changing one Model. Creating a new App through AI REST is not supported because AI tokens can select only existing Apps.

## Table

`fields` is required and may be empty structurally. `titleField` and every indexed field must exist. Field names are unique and cannot use framework system fields. Enum and reference targets must exist.

```json
{
  "kind": "table",
  "name": "SALES_Order",
  "app": "sales",
  "model": "Core",
  "layer": "ISV",
  "label": "Sales orders",
  "titleField": "orderNo",
  "fields": [
    { "name": "orderNo", "type": "string", "mandatory": true, "maxLength": 30 },
    { "name": "amount", "type": "real", "default": 0 }
  ],
  "indexes": [
    { "name": "SALES_Order_OrderNoIdx", "fields": ["orderNo"], "unique": true }
  ]
}
```

Schema synchronization is additive. Removing metadata preserves physical data; changing existing SQLite structures requires a migration plan.

## Enum

`values` is required. Each value requires an identifier `name` and integer `value`; numeric values should remain stable after deployment.

```json
{
  "kind": "enum",
  "name": "SALES_OrderStatus",
  "app": "sales",
  "model": "Core",
  "layer": "ISV",
  "label": "Order status",
  "values": [
    { "name": "Open", "value": 0, "label": "Open" },
    { "name": "Confirmed", "value": 1, "label": "Confirmed" }
  ]
}
```

## Form

`table` is required and must exist. Every field in lists/groups and every line, Chart, Report, Function, Picker, and Privilege reference is validated.

```json
{
  "kind": "form",
  "name": "SALES_OrderForm",
  "app": "sales",
  "model": "Core",
  "layer": "ISV",
  "label": "Sales orders",
  "table": "SALES_Order",
  "listFields": ["orderNo", "amount"],
  "filterFields": ["orderNo"],
  "groups": [
    { "id": "order-general", "label": "General", "fields": ["orderNo", "amount"] }
  ]
}
```

Give stable `id` values to groups, Charts, lines, and actions that a higher Layer may override.

## Menu

`items` is required. Each target must resolve to an existing Form, Function, or Report. Groups contain nested `items`.

```json
{
  "kind": "menu",
  "name": "SALES_MainMenu",
  "app": "sales",
  "model": "Core",
  "layer": "ISV",
  "label": "Sales navigation",
  "items": [
    {
      "id": "sales-orders",
      "label": "Sales orders",
      "icon": "grid",
      "visible": true,
      "target": { "type": "form", "name": "SALES_OrderForm" }
    }
  ]
}
```

## Privilege

All grant collections are optional, so an empty Privilege is structurally valid but grants nothing. Table permissions are operation-specific.

```json
{
  "kind": "privilege",
  "name": "SALES_OrderMaintainPrivilege",
  "app": "sales",
  "model": "Core",
  "layer": "ISV",
  "label": "Maintain sales orders",
  "tablePermissions": [
    { "table": "SALES_Order", "read": true, "create": true, "update": true, "delete": false }
  ],
  "forms": ["SALES_OrderForm"]
}
```

## Duty

`privileges` is required and every name must resolve.

```json
{
  "kind": "duty",
  "name": "SALES_OrderProcessingDuty",
  "app": "sales",
  "model": "Core",
  "layer": "ISV",
  "label": "Process sales orders",
  "privileges": ["SALES_OrderMaintainPrivilege"]
}
```

## Role

`duties` and `privileges` are both optional. Use Duties for reusable groupings and direct Privileges only when appropriate.

```json
{
  "kind": "role",
  "name": "SALES_OrderClerkRole",
  "app": "sales",
  "model": "Core",
  "layer": "ISV",
  "label": "Order clerk",
  "duties": ["SALES_OrderProcessingDuty"]
}
```

Role metadata does not assign users or App Access; administrators perform those assignments separately.

## Script

`code` is required. A Script registers lifecycle behavior and actions. It is executable server-side code and requires human security review.

```json
{
  "kind": "script",
  "name": "SALES_OrderValidationScript",
  "app": "sales",
  "model": "Core",
  "layer": "ISV",
  "label": "Order validation",
  "code": "// Register reviewed hooks and events here."
}
```

See [Develop Scripts](scripts.md) for the execution contract.

## Function

`code` is required. `executionMode` is `transactional` (default) or `async`. `privileges` may name Privileges required to invoke it.

```json
{
  "kind": "function",
  "name": "SALES_RecalculateOrder",
  "app": "sales",
  "model": "Core",
  "layer": "ISV",
  "label": "Recalculate order",
  "executionMode": "transactional",
  "privileges": ["SALES_OrderMaintainPrivilege"],
  "code": "return { ok: true };"
}
```

Async Functions may await bounded HTTP/email services but must use explicit short transactions for database writes.

## Report

`dataSource` and `bands` are required. The source Table, field elements, parameters, Line source relationships, and Tablix columns are validated.

```json
{
  "kind": "report",
  "name": "SALES_OrderListReport",
  "app": "sales",
  "model": "Core",
  "layer": "ISV",
  "label": "Order list",
  "dataSource": "SALES_Order",
  "defaultFont": "Noto Sans Thai",
  "page": { "size": "A4", "orientation": "portrait", "margins": [40, 40, 40, 40] },
  "bands": [
    {
      "kind": "detail",
      "layout": "tablix",
      "height": 20,
      "elements": [],
      "tablix": {
        "columns": [
          { "field": "orderNo", "label": "Order", "width": 140 },
          { "field": "amount", "label": "Amount", "align": "right" }
        ]
      }
    }
  ]
}
```

Tablix is allowed only on Detail bands. See [Design paginated Reports](reports.md).

## View

`source` and at least one output `column` are required. Field references use `alias.field`. Framework tables cannot be queried. Cross-App source Tables require an App dependency.

```json
{
  "kind": "view",
  "name": "SALES_OrderSummaryView",
  "app": "sales",
  "model": "Core",
  "layer": "ISV",
  "label": "Order summary",
  "source": { "table": "SALES_Order", "alias": "o" },
  "columns": [
    { "name": "orderNo", "expression": { "type": "field", "ref": "o.orderNo" } },
    { "name": "amount", "expression": { "type": "field", "ref": "o.amount" } }
  ],
  "orderBy": [{ "column": "orderNo", "direction": "asc" }]
}
```

## Chart

`type`, `view`, and at least one measure are required. Non-KPI Charts also require a valid `dimension`; KPI requires exactly one measure.

```json
{
  "kind": "chart",
  "name": "SALES_OrderAmountChart",
  "app": "sales",
  "model": "Core",
  "layer": "ISV",
  "label": "Order amounts",
  "type": "bar",
  "view": "SALES_OrderSummaryView",
  "dimension": "orderNo",
  "measures": [
    { "field": "amount", "label": "Amount", "color": "#2563eb" }
  ],
  "legend": true,
  "stacked": false
}
```

Chart types are `bar`, `line`, `pie`, `donut`, and `kpi`.

# Extension kinds

All Extensions require common placement, a target property, a Layer strictly higher than the target, and a name ending `_Extension`. Cross-App targets require `dependsOn`. Only one Extension of the same kind may target the same base from one App/Model.

The canonical name is generated from App prefix, a normalized Model name, and the target name. The target is preserved. For example, App `sales`, Model `Customizations`, and target `SALES_Order` produce `SALES_Customizations_SALES_Order_Extension`.

## Table Extension

Target: `table`. Deltas: `fields`, `indexes`, `fieldOverrides`. Added field names must not already exist.

```json
{
  "kind": "tableExtension",
  "name": "SALES_Customizations_SALES_Order_Extension",
  "app": "sales",
  "model": "Customizations",
  "layer": "CUS",
  "table": "SALES_Order",
  "fields": [{ "name": "customerReference", "type": "string", "maxLength": 40 }],
  "fieldOverrides": [{ "field": "amount", "label": "Net amount", "allowEdit": false }]
}
```

## Form Extension

Target: `form`. Additive collections append new content. `elementOverrides` changes presentation by stable ID; `lineOverrides` replaces selected inherited Line presentation, fields, aggregates, or actions without changing `table`/`refField`.

```json
{
  "kind": "formExtension",
  "name": "SALES_Customizations_SALES_OrderForm_Extension",
  "app": "sales",
  "model": "Customizations",
  "layer": "CUS",
  "form": "SALES_OrderForm",
  "listFields": ["customerReference"],
  "elementOverrides": [
    { "targetId": "order-general", "label": "Order details", "order": 10 }
  ],
  "lineOverrides": []
}
```

Every override `targetId` must exist in the effective inherited Form.

## Menu Extension

Target: `menu`. Add root items with `items`; attach below an inherited group with `parentId`; use `itemOverrides` for inherited items. `insertions` with stable-ID paths remains supported for compatibility.

```json
{
  "kind": "menuExtension",
  "name": "SALES_Customizations_SALES_MainMenu_Extension",
  "app": "sales",
  "model": "Customizations",
  "layer": "CUS",
  "menu": "SALES_MainMenu",
  "items": [
    {
      "id": "sales-custom-report",
      "label": "Order list report",
      "target": { "type": "report", "name": "SALES_OrderListReport" }
    }
  ],
  "itemOverrides": [
    { "targetId": "sales-orders", "label": "Orders", "visible": true, "order": 5 }
  ]
}
```

## Enum Extension

Target: `enum`. `values` is required even when it is `[]`. New numeric values must not destabilize existing meanings; `valueOverrides` relabels an inherited value by name.

```json
{
  "kind": "enumExtension",
  "name": "SALES_Customizations_SALES_OrderStatus_Extension",
  "app": "sales",
  "model": "Customizations",
  "layer": "CUS",
  "enum": "SALES_OrderStatus",
  "values": [{ "name": "OnHold", "value": 2, "label": "On hold" }],
  "valueOverrides": [{ "name": "Confirmed", "label": "Released" }]
}
```

## Privilege Extension

Target: `privilege`. Permission collections are merged and deduplicated.

```json
{
  "kind": "privilegeExtension",
  "name": "SALES_Customizations_SALES_OrderMaintainPrivilege_Extension",
  "app": "sales",
  "model": "Customizations",
  "layer": "CUS",
  "privilege": "SALES_OrderMaintainPrivilege",
  "reports": ["SALES_OrderListReport"],
  "views": ["SALES_OrderSummaryView"]
}
```

## Duty Extension

Target: `duty`. `privileges` is optional and appended uniquely.

```json
{
  "kind": "dutyExtension",
  "name": "SALES_Customizations_SALES_OrderProcessingDuty_Extension",
  "app": "sales",
  "model": "Customizations",
  "layer": "CUS",
  "duty": "SALES_OrderProcessingDuty",
  "privileges": ["SALES_OrderMaintainPrivilege"]
}
```

## Role Extension

Target: `role`. Add Duties and/or Privileges.

```json
{
  "kind": "roleExtension",
  "name": "SALES_Customizations_SALES_OrderClerkRole_Extension",
  "app": "sales",
  "model": "Customizations",
  "layer": "CUS",
  "role": "SALES_OrderClerkRole",
  "duties": ["SALES_OrderProcessingDuty"]
}
```

## Script Extension

Target: `script`; `code` is required. The resulting executable composition must be reviewed.

```json
{
  "kind": "scriptExtension",
  "name": "SALES_Customizations_SALES_OrderValidationScript_Extension",
  "app": "sales",
  "model": "Customizations",
  "layer": "CUS",
  "script": "SALES_OrderValidationScript",
  "code": "// Register reviewed additional behavior here."
}
```

## Function Extension

Target: `function`; `code` is required. The Chain-of-Command body receives `next`; call it deliberately when the base implementation must continue.

```json
{
  "kind": "functionExtension",
  "name": "SALES_Customizations_SALES_RecalculateOrder_Extension",
  "app": "sales",
  "model": "Customizations",
  "layer": "CUS",
  "function": "SALES_RecalculateOrder",
  "code": "const result = next(args); return { ...result, customized: true };"
}
```

## View Extension

Target: `view`. Joins, columns, filters, and ordering are appended; `columnOverrides` relabels an existing output column. The complete effective View must still pass grouping, type, App-dependency, and protected-table rules.

```json
{
  "kind": "viewExtension",
  "name": "SALES_Customizations_SALES_OrderSummaryView_Extension",
  "app": "sales",
  "model": "Customizations",
  "layer": "CUS",
  "view": "SALES_OrderSummaryView",
  "columns": [
    { "name": "customerReference", "expression": { "type": "field", "ref": "o.customerReference" } }
  ],
  "columnOverrides": [
    { "column": "amount", "label": "Net amount" }
  ]
}
```

## Chart Extension

Target: `chart`. Add measures, set `legend`/`stacked`/`label`, or override an existing measure by `field`. The effective Chart must still satisfy View output and KPI rules.

```json
{
  "kind": "chartExtension",
  "name": "SALES_Customizations_SALES_OrderAmountChart_Extension",
  "app": "sales",
  "model": "Customizations",
  "layer": "CUS",
  "chart": "SALES_OrderAmountChart",
  "legend": false,
  "measureOverrides": [
    { "field": "amount", "label": "Net amount", "color": "#16a34a" }
  ]
}
```

## Validation checklist

Before applying any Artifact:

1. Confirm identity, App prefix, existing Model, and effective Layer.
2. Confirm every referenced object exists in the same candidate workspace or a declared dependency.
3. Confirm field, parameter, View output, and binding types match.
4. Confirm Extension target Layer is lower and stable IDs exist.
5. Review Scripts and Functions as untrusted executable code.
6. Validate a complete ChangeSet and review `diff`, `schemaEffects`, warnings, and high-risk flags.
7. Apply only with the intended signed-in reviewer and a fresh revision.

## Related topics

[Artifact API](artifact-api.md) · [Nested structures](artifact-components.md) · [Metadata](metadata.md) · [Extensions](extensions.md) · [AI REST API](ai-rest-api.md)
