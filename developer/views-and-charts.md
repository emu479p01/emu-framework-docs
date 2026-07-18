# Build Views and embedded Charts

## Purpose

Define a reusable, validated query as a View, visualize it with a Chart artifact, and embed the Chart in a generated Form.

## View artifacts

A View is a virtual query that runs when its schema, data, or export endpoint is called. It is not a materialized SQLite view and does not accept raw SQL, subqueries, window functions, or computed expressions.

View metadata supports:

- one source table and alias
- `inner` and `left` joins using field equality
- field and aggregate output columns
- typed `string`, `int`, `real`, `boolean`, `date`, and `datetime` parameters
- literal or parameter filters
- grouping, sorting, and `count`, `sum`, `avg`, `min`, and `max`

```json
{
  "kind": "view",
  "name": "SALES_CustomerTotalsView",
  "label": "Customer totals",
  "app": "sales",
  "model": "Analytics",
  "layer": "DEV",
  "source": { "table": "SALES_Order", "alias": "o" },
  "joins": [
    {
      "type": "inner",
      "table": "SALES_Customer",
      "alias": "c",
      "on": [{ "left": "o.customerId", "right": "c.id" }]
    }
  ],
  "parameters": [{ "name": "fromDate", "type": "date", "required": true }],
  "filters": [
    { "ref": "o.orderDate", "operator": "gte", "value": { "parameter": "fromDate" } }
  ],
  "columns": [
    { "name": "customer", "expression": { "type": "field", "ref": "c.name" } },
    { "name": "total", "expression": { "type": "aggregate", "fn": "sum", "ref": "o.amount" } }
  ],
  "groupBy": ["c.name"],
  "orderBy": [{ "column": "total", "direction": "desc" }]
}
```

Field references use the declared alias and field name, such as `o.orderDate`. Output sort entries reference output-column names. Designer/change-set validation checks table and field existence, types, grouping, App dependencies, aliases, joins, and protected tables. Runtime compilation quotes identifiers and binds values as parameters.

## View authorization

For an interactive user, all of these must allow the request:

1. `FW_AppAccess.canOpen=true` for the App that owns the View.
2. A Role/Duty/Privilege that includes the View name in `views`.
3. Read permission for every source and joined table.
4. Every row scope applied to those source tables.

The same checks apply to the schema, JSON data, and CSV export endpoints. A scoped service token is the separate integration path described in [Connect Power BI to the View API](../admin/power-bi-view-api.md).

## Chart artifacts

A Chart references one View and reuses it in one or more Forms. Supported types are `bar`, `line`, `pie`, `donut`, and `kpi`.

```json
{
  "kind": "chart",
  "name": "SALES_CustomerTotalsChart",
  "label": "Sales by customer",
  "app": "sales",
  "model": "Analytics",
  "layer": "DEV",
  "type": "bar",
  "view": "SALES_CustomerTotalsView",
  "dimension": "customer",
  "measures": [{ "field": "total", "label": "Sales", "color": "#2563eb" }],
  "legend": true,
  "stacked": false
}
```

Non-KPI Charts require a dimension from the View output. A KPI requires exactly one measure. Chart authorization is inherited from the referenced View privilege and source-table read permissions.

## Embed a Chart in a Form

Add `charts` to a Form or Form Extension. Each entry chooses a `half` or `full` width and binds View parameters from the current record or a literal.

```json
{
  "charts": [
    {
      "chart": "SALES_CustomerTotalsChart",
      "width": "full",
      "parameterBindings": [
        { "parameter": "fromDate", "source": "literal", "value": "2026-01-01" }
      ]
    }
  ]
}
```

Use `{ "source": "record", "field": "customerId" }` when a View parameter should come from a current-record field. Embedded Charts render after Form groups and before line grids. A new record that does not yet supply a required record-bound parameter asks the user to save first. Unauthorized Charts are omitted from Form metadata, and direct View or Chart calls return an authorization error.

Apache ECharts is bundled locally; no CDN is required. The generated component resizes with its container and disposes its ECharts instance when removed.

## Related topics

[Metadata](metadata.md) · [Security](security.md) · [Web Designer](../user/web-designer.md) · [Power BI](../admin/power-bi-view-api.md)
