# Use the Web Designer

## Purpose

Create or customize metadata-driven apps from the browser.

## Audience

Application customizers and developers.

## Prerequisites

Your account needs Designer permission for the target app.

## Procedure

1. Open **Web Designer** and select an existing app or create one.
2. Use **Simple Builder** to create a table, form, and menu entry together.
3. Add fields and validate their names and types.
4. Save, review the generated change set, and apply it.
5. Open the app and verify the form and list.

## Recommended customization order

Work through objects in this order so each step can reference the ones before it: **App → Model → Object → Menu → Privilege/Duty/Role**.

The Designer supports these object kinds, including Extensions of the kinds that support extension: App, Table, Enum, Form, Menu, Function, Script, Report, Privilege, Duty, Role.

## Tables and fields

For each field, set its type, label, required flag, read-only flag, and whether it can be edited on create versus update. A reference field also sets the related table, display fields, on-delete behavior, and copy fields.

### Dynamic lookup filters

A reference field's lookup filter can pull its comparison value from three sources:

| Source | Description | Example |
| --- | --- | --- |
| Constant | A fixed value | `status eq 0` |
| Current record field | A field from the same record or line | `categoryId` |
| Field from selected lookup | A field from the record already selected in another reference on the same form | Select `customerId`, then filter the next lookup using `customer.groupId` |

When the source field changes, any dropdown that depends on it reloads its options.

```json
{
  "name": "itemId",
  "type": "reference",
  "reference": {
    "table": "SALES_Item",
    "displayFields": ["itemNo", "name"],
    "filters": [
      {
        "field": "categoryId",
        "operator": "eq",
        "value": { "source": "record", "field": "categoryId" }
      },
      {
        "field": "status",
        "operator": "eq",
        "value": { "source": "lookup", "field": "customerId", "lookupField": "allowedItemStatus" }
      }
    ]
  }
}
```

## Forms

- `listFields` sets the columns shown on the list page.
- `filterFields` sets the columns a user can search by; when omitted, the form falls back to `listFields`.
- `groups` arranges fields on the detail page.
- `lines` builds a master-detail grid with aggregates and line-level actions.
- A header action that should appear before the record is saved for the first time needs `showOnCreate: true`.

```json
{
  "kind": "form",
  "name": "SALES_OrderForm",
  "table": "SALES_Order",
  "listFields": ["orderNo", "customerId", "status"],
  "filterFields": ["orderNo", "customerId", "status"],
  "actions": [
    {
      "label": "Calculate defaults",
      "type": "function",
      "target": "SALES_CalculateDefaults",
      "showOnCreate": true
    }
  ]
}
```

A Function that runs before create does not receive `recordId`, but it still receives the current `record` values — write it to handle the missing-ID case. See [Develop Functions and actions](../developer/functions.md).

## Function execution mode

Choose **Transactional — synchronous and atomic** for normal database operations. This is the default and wraps the Function in one transaction.

Choose **Async integration — supports await, HTTP and email** when the Function calls `services.http.request(...)` or `services.email.send(...)`. Async mode does not keep one database transaction open while waiting for the network. Put database changes in short explicit `ctx.tts()` blocks, and never await an external request inside a transaction.

The Function body receives `ctx`, `args`, `kernel`, and `services`. Async Functions must handle rejected requests and non-success HTTP status codes explicitly.

## Report Designer on small screens

The report canvas keeps its document width so element positions remain stable. On a phone or narrow browser, scroll horizontally inside the canvas. Settings, parameters, line sources, and the selected-element panel stack vertically.

`Noto Sans Thai` is bundled and available without a Google Fonts API key. PDF output automatically selects it for Thai text.

## Menus and the sidebar

- Level 1 menu items are the primary sidebar entries.
- Level 2 and deeper menu items open as a navigation overlay to the right of the sidebar; the overlay does not shrink the content area and closes when the user selects a menu item, clicks outside it, presses Escape, or uses its close button.
- A menu target can be a Form, Function, Report, route, or a group/submenu.

## Extensions in the Designer

Use an Extension when you need to add a field, form layout/action, menu item, enum value, or security metadata without changing the base object. The source layer must be strictly higher than the target layer, and extending across apps requires that app dependency to be declared. Use the Designer-generated Extension name to avoid duplicate names and to keep the target traceable. See [Create extensions](../developer/extensions.md) and [Work with metadata layers](../developer/layers.md).

## Expected result

Metadata is stored in `designer.db`; additive schema changes are applied to the business database.

## Common errors

- Names are identifiers: use letters, numbers, and underscores without spaces.
- Destructive schema changes and native server code still require a developer.

## Related topics

[Metadata](../developer/metadata.md) · [Extensions](../developer/extensions.md) · [Functions and actions](../developer/functions.md) · [Customization checklist](../developer/customization-checklist.md) · [Backup](../admin/backup.md)
