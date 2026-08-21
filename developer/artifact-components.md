# Nested metadata structures

## Purpose

Define reusable nested objects used inside Table, Form, Menu, Report, View, Chart, and Extension Artifacts.

Most nested schemas reject unlisted properties; use only the properties documented here. “Required” refers to the nested object itself; the parent collection may still be optional.

## Table field

Required: `name`, `type`.

| Property | Type | Meaning |
| --- | --- | --- |
| `name` | identifier | Field name; cannot be `id`, `createdAt`, `createdBy`, `modifiedAt`, or `modifiedBy`. |
| `type` | enum | `string`, `int`, `real`, `boolean`, `date`, `datetime`, `enum`, or `reference`. |
| `label` | string | User-facing label. |
| `mandatory` | boolean | Required business value. Enum and read-only fields cannot be mandatory. |
| `readOnly` | boolean | Generated Forms and REST writes cannot edit it; trusted code may. |
| `allowEdit` | boolean | Allow edit after creation. |
| `allowEditOnCreate` | boolean | Allow edit during creation. |
| `maxLength` | integer ≥ 1 | String length limit. |
| `enumName` | identifier | Required semantically when `type` is `enum`; must reference an existing Enum. |
| `reference` | object | Required semantically when `type` is `reference`. |
| `default` | scalar or null | String, number, boolean, or null. |

```json
{
  "name": "customerId",
  "type": "reference",
  "label": "Customer",
  "mandatory": true,
  "reference": {
    "table": "SALES_Customer",
    "displayFields": ["accountNo", "name"],
    "onDelete": "restrict"
  }
}
```

### Reference settings

Required: `table`.

- `displayField` selects one display field; `displayFields` selects one or more and takes the practical multi-field form.
- `onDelete` is `restrict`, `cascade`, or `setNull`. A mandatory reference cannot use `setNull`.
- `copyFields` contains `{ "from": "sourceField", "to": "localField" }`. `from` must exist on the referenced table; `to` must be a different local field.
- `filters` contains `field`, `operator`, and `value`. Operators are `eq`, `ne`, `gt`, `gte`, `lt`, `lte`, and `contains`.
- A filter value may be a literal/null, `{ "source": "record", "field": "localField" }`, or `{ "source": "lookup", "field": "localReference", "lookupField": "fieldOnItsTarget" }`.

## Index

Required: `name`, non-empty `fields`. Optional: `unique`.

Every indexed field must exist on the effective Table.

```json
{ "name": "SALES_Order_OrderNoIdx", "fields": ["orderNo"], "unique": true }
```

## Form group

Required: `fields`. Optional: `id`, `label`, `hidden`, `order`.

All fields must exist on the Form's Table. Give `id` a stable value when an Extension may override the group.

## Form action

Required: `label` and a semantic target.

| Property | Meaning |
| --- | --- |
| `id`, `hidden`, `order` | Stable presentation identity and state. |
| `type` | `function`, `report`, or `picker`; defaults to `function`. |
| `target` | Required function/report name. For a picker, still provide a stable target name used by the action. |
| `action` | Legacy function target; new metadata uses `type` + `target`. |
| `privilege` | Optional named Privilege for the action. |
| `showOnCreate` | Show before the record has an ID. |
| `picker` | Required when `type` is `picker`. |

A picker requires `table` and non-empty `columns`. It may include `searchFields`, `multiple`, `allocation { availableField, quantityLabel? }`, and filters. Picker filter values may be literals/null or `{ "source": "record" | "line", "field": "..." }`; `line` is valid only on a line action.

## Form Line grid

Required: `table`, `refField`, `fields`. Optional: `id`, `label`, `hidden`, `order`, `aggregates`, `actions`.

`refField` is the child Table field pointing to the parent record. Do not include `refField` as an editable Line column. Aggregates are:

```json
[
  { "fn": "count", "label": "Lines" },
  { "fn": "sum", "field": "amount", "label": "Total" },
  { "fn": "avg", "field": "quantity", "label": "Average" }
]
```

`sum` and `avg` require an `int` or `real` field. `count` does not require `field`.

## Embedded Form Chart

Required: `chart`. Optional: `id`, `label`, `hidden`, `order`, `width`, `parameterBindings`.

`width` is `half` or `full`. Each binding requires `parameter` and `source`:

- `source: "record"` requires `field` with a compatible type.
- `source: "literal"` requires `value` (string, number, boolean, or null).
- Every required View parameter must be bound exactly once.

## Menu item

All properties are structurally optional, but a useful leaf needs a label and target. Prefer:

```json
{
  "id": "sales-orders",
  "label": "Sales orders",
  "icon": "grid",
  "visible": true,
  "order": 10,
  "target": { "type": "form", "name": "SALES_OrderForm" }
}
```

Targets are `{ "type": "group" }` or `{ "type": "form" | "function" | "report", "name": "..." }`. `form`, `route`, and `action` are legacy-compatible shortcuts. `items` contains nested menu items. In Menu Extensions, `parentId` attaches a new item under an inherited group. Allowed icons are `app`, `grid`, `users`, `settings`, `database`, `table`, `chart`, `shield`, `wrench`, and `file`.

Use `visible`; `hidden` remains compatible, and `visible` wins when both are present.

## Security permission structures

Table permission requires `table`; optional booleans are `read`, `create`, `update`, and `delete`. Referenced Tables, Forms, Functions, Reports, Views, Privileges, Duties, and Roles must exist in the candidate workspace.

## Report page, bands, and elements

Page settings:

```json
{
  "size": "A4",
  "orientation": "portrait",
  "margins": [40, 40, 40, 40]
}
```

The schema accepts a four-number tuple. Runtime type documentation defines the order as `[top, right, bottom, left]` in points.

A band requires `kind`, `height`, and `elements`:

- `kind`: `header`, `detail`, or `footer`; `pageHeader`/`pageFooter` are legacy-compatible.
- `displayOn`: `firstPage`, `everyPage`, or `lastPage`, only for Header/Footer.
- `layout`: `freeform` or `tablix`.
- A Tablix is valid only on a Detail band, requires `tablix`, and requires `elements: []`.

A freeform element requires `id`, `type`, `x`, `y`, `width`, and `height`. `type` is `text`, `field`, `image`, `line`, or `rect`; provide `text` for text and `field` for bound fields. Optional `format` and style properties are `fontSize`, `bold`, `italic`, `fontFamily`, `align`, `color`, and `borderWidth`.

A Tablix requires at least one column. Each column requires `field`; optional properties are `label`, positive `width`, `align`, and `format`. Optional Tablix properties are positive `headerHeight`, positive `rowHeight`, `headerStyle`, `rowStyle`, and `border`.

Line sources require `table`, `refField`, and `bands`. Parameters require `field`; optional `operator` is `eq`, `from`, or `to`, with optional `label` and `required`.

## View structures

Source requires `table` and `alias`. A Join requires `type` (`inner` or `left`), `table`, unique `alias`, and non-empty `on` conditions containing `left` and `right` references in `alias.field` form.

A View column requires `name` and `expression`:

```json
{ "name": "customerName", "expression": { "type": "field", "ref": "c.name" } }
```

```json
{ "name": "totalAmount", "expression": { "type": "aggregate", "fn": "sum", "ref": "o.amount" } }
```

Aggregates are `count`, `sum`, `avg`, `min`, and `max`. Only `count` may omit `ref`; `sum`/`avg` require numeric fields. If any output is aggregated, every non-aggregate field expression must appear in `groupBy`.

A View parameter requires `name` and `type` (`string`, `int`, `real`, `boolean`, `date`, or `datetime`); `required` is optional. A filter requires `ref`, `operator`, and `value`. Operators add `in` to the normal comparison set. `value` is a literal/null, literal array, or `{ "parameter": "name" }`. `contains` requires a string field; `in` requires an array or parameter.

`orderBy` items require an output `column`; `direction` is `asc` or `desc`.

## Chart measure

Required: `field`. Optional: `label`, `color`. `field` must be an output column of the Chart's View.

## Extension overrides

- `fieldOverrides`: `field` plus optional `label`, `readOnly`, `allowEdit`, `allowEditOnCreate`.
- `elementOverrides`: `targetId` plus optional `label`, `hidden`, `order`.
- `lineOverrides`: `targetId` plus optional `label`, `hidden`, `order`, `fields`, `aggregates`, `actions`. It cannot change the inherited relationship identity (`table`/`refField`).
- `itemOverrides`: `targetId` plus optional `label`, icon, visibility, order, or target.
- `valueOverrides`: `name` and required replacement `label`.
- `columnOverrides`: `column` and required replacement `label`.
- `measureOverrides`: `field` plus optional `label` and `color`.

## Related topics

[Artifact API](artifact-api.md) · [Artifact kinds](artifact-types.md) · [Forms](../user/web-designer.md) · [Reports](reports.md) · [Views and Charts](views-and-charts.md)
