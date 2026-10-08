# Use the Web Designer

## Purpose

Create or customize metadata-driven apps from the browser.

## Audience

Application customizers and developers.

## Prerequisites

Your account needs Designer permission for the target app.

## Procedure

1. Open **Web Designer** and select an existing App or create one. A new App starts with zero Models.
2. Add a Model and choose its Layer, or use the explicit Model step in **Simple Builder**. No Model or Layer is selected automatically.
3. Select the App and Model before creating an artifact.
4. Use **Simple Builder** to create a table, Form, and menu entry together.
5. Add fields and validate their names and types.
6. Save, review the generated change set, and apply it.
7. Open the App and verify the Form and list with a separately authorized runtime account.

Artifact lists are paginated and filterable by App, Model, and kind. In a large workspace, move through every page or narrow the filters instead of assuming the first page contains every Artifact.

## Recommended customization order

Work through objects in this order so each step can reference the ones before it: **App → Model → Object → Menu → Privilege/Duty/Role**.

The Designer supports these object kinds, including Extensions of the kinds that support extension: App, Table, Enum, Form, Menu, Function, Script, Report, View, Chart, Privilege, Duty, Role, Translation, and Data Entity. Translation, Data Entity, and Data Entity Extension are created from the **New** menu like other artifacts.

`canCustomize` grants Designer capability for the selected App only. It does not grant `canOpen` or business-data permissions. System Administrators can inspect Framework metadata under **Framework — Read-only**, but nobody can change, delete, package, or extend it.

## Tables and fields

For each field, set its type, label, required flag, read-only flag, and whether it can be edited on create versus update. Enum and read-only fields must be optional. Trusted Functions and Scripts can populate read-only fields, but REST writes and generated Forms cannot edit them. A reference field also sets the related table, display fields, on-delete behavior, and copy fields.

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
- `charts` embeds reusable Chart artifacts after groups and before line grids.
- `lines` builds a master-detail grid with aggregates and line-level actions.
- Line create/update/delete operations show a confirmation dialog before committing.
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

When a user opens **New**, the form is a draft: default values and the results of `initValue` hooks, including read-only document numbers, are shown before the record is saved, and nothing is inserted until the user saves. In v1.4.0 the metadata validator still rejects a field that is both read-only and mandatory, so keep read-only fields optional and let `initValue` supply them. Hooks (`initValue`, `validateWrite`, `validateDelete`) and data event handlers must be synchronous; move awaited or external work into an async Function. A draft is rejected with `409` if table, Script, or Function metadata changes while it is open, so users reopen **New** after you apply a change. See [Record lifecycle](../developer/record-lifecycle.md).

A Function that runs before create does not receive `recordId`, but it still receives the current `record` values — write it to handle the missing-ID case. See [Develop Functions and actions](../developer/functions.md).

## Function execution mode

Choose **Transactional — synchronous and atomic** for normal database operations. This is the default and wraps the Function in one transaction.

Choose **Async integration — supports await, HTTP and email** when the Function calls `services.http.request(...)` or `services.email.send(...)`. Async mode does not keep one database transaction open while waiting for the network. Put database changes in short explicit `ctx.tts()` blocks, and never await an external request inside a transaction.

The Function body receives `ctx`, `args`, `kernel`, and `services`. Async Functions must handle rejected requests and non-success HTTP status codes explicitly.

## Function image input

Select **Image input** on a Function when it should receive photos or image files for an existing record. Then choose the **Attachment table** (a business table, not a `FW_*` table), the **Record ID argument** that carries the record ID, and whether to **Allow multiple images**. When a user runs the Function, its dialog offers **Choose images** and **Take a photo** (JPEG, PNG, or WebP); the images are uploaded as attachments of the record and the Function receives their `attachmentIds`. The Function's code is never sent to the browser. See [Work with attachments](attachments.md) for what users see and [Attachments and Function image input](../developer/attachments.md) for the code contract.

## Translations and the default language

Use a Translation artifact to translate labels for one locale.

1. Choose **New → Translation** with the App and Model selected.
2. Set the **Locale**, for example `th` or `th-TH`.
3. In the editor, each row is a translatable target: its **Resource key**, **Default text**, and your **Translation**. Search the list, use **Show untranslated only**, or filter by category. **Copy default texts** pre-fills the empty rows so you can edit them.
4. Leave a translation cell empty when you do not want to override the label. A partial translation only changes the keys it contains.
5. Save. Users who selected that locale see the new labels.

Targets that no longer exist in the metadata are listed as **Saved keys without a current metadata target**. Resource keys beginning with `ui.` belong to the framework and are read-only. If the same key is translated in several Layers, the higher Layer wins.

Each App has a **Default language**, set by editing the App (default `en`). A user sees labels in this order: the user's exact locale, its base language (`th` for `th-TH`), the App's default language, and then the label stored in the metadata. The Designer warns when no translation exists yet for the default language you chose. Business data values, script code, technical names, and raw server errors are never translated. See [Localize labels with Translations](../developer/localization.md).

## Data Entities and Data Entity Extensions

A Data Entity describes a business document (a header and its lines) that users can import and export, and that administrators can archive.

In the **Data Entity** editor, set:

- **Root table** and **Business key**: the fields that identify one document.
- **Fields** exported from the root table.
- **Lines**: use **+ Add line** for each line table and its parent reference.
- **Archive eligible** and **Business date field** if the document can be archived. A business date field is required when archiving is allowed.

A **Data Entity Extension** adds to an existing Data Entity. Choose the **Data Entity**, then use **Additional root fields** and **Additional fields on existing lines**, or add new lines. An extension cannot change the root table, the business key, the line relationships, or the archive settings. Import, export, archive, and restore all read the merged entity. The editor checks the structure on the client, and the server validation has the final say when you save. See [Exchange documents with Data Entities](../developer/data-entities.md) and [Manage storage and archiving](../admin/storage-and-archive.md).

## Models, ISV license vendor, and deployment packages

When you create or edit a Model on the ISV Layer, the Model dialog shows an optional **License vendor ID**. Leave it blank for an unlicensed Model. Entering a vendor ID makes the Model require an offline license from that vendor; an administrator then registers the vendor and imports the license (see [Manage Apps, Models, deployments, and licenses](../admin/apps-models-licenses.md)). Once a license requirement is installed, ordinary edits cannot remove or change it.

To move whole Models to another environment:

1. Open the App's menu and choose **Build deployment package**.
2. Choose **Vendor update — SYS / LOC / ISV** or **UAT → Prod — all layers**, then select complete Models.
3. Choose **Download package**.
4. At the destination, import the package. **Review Metadata Import** shows a preview that starts collapsed: per Model it lists create, update, and delete counts and the number of high-risk items. Expand a Model, or use **Expand all**, to see each artifact and where moved or deleted artifacts came from.
5. Confirm only after you have reviewed the high-risk items. Missing dependencies, ownership conflicts, and incompatible customizations block the import.

Selected Models replace their metadata at the destination; other Models and all business data are kept. File-based Apps and the framework's `system` App cannot be packaged. See [Deploy Models and issue ISV licenses](../developer/model-deployment.md) for the package format.

## Report Designer on small screens

The report canvas keeps its document width so element positions remain stable. On a phone or narrow browser, scroll horizontally inside the canvas. Settings, parameters, line sources, and the selected-element panel stack vertically.

`Noto Sans Thai` is bundled and available without a Google Fonts API key. PDF output automatically selects it for Thai text.

Detail and Line bands can use either Freeform or Tablix layout. Tablix provides columns, formatting, header/row styles, repeated headers, and native page planning. Header and Footer bands can render on the first, every, or last page. See [Design paginated Reports](../developer/reports.md).

## Menus and the sidebar

- Level 1 menu items are the primary sidebar entries.
- Level 2 and deeper menu items open as a navigation overlay to the right of the sidebar; the overlay does not shrink the content area and closes when the user selects a menu item, clicks outside it, presses Escape, or uses its close button.
- A menu target can be a Form, Function, Report, route, or a group/submenu.

## Views and Charts

Use the View builder to define a declarative query from validated tables, joins, fields, parameters, filters, grouping, and aggregates. Use the Chart editor to map View output to a bar, line, pie, donut, or KPI visualization. Then use the Form Chart editor to choose width and bind View parameters from current-record fields or literals.

Designer validates references, types, grouping, App dependencies, and protected tables before apply. See [Build Views and embedded Charts](../developer/views-and-charts.md).

## Extensions in the Designer

Use an Extension when you need to add or adjust metadata without changing the base object. The Designer shows the inherited customization chain as read-only and saves only the current Model/Layer delta. Table/Enum/Form/Menu Extensions support presentation overrides, while View, Chart, and Function Extensions add query, visualization, or Chain-of-Command behavior.

The source layer must be strictly higher than the target layer, and extending across apps requires that app dependency to be declared. Use the Designer-generated canonical Extension name so overrides survive lower-layer changes and duplicate targets are rejected. Form Extensions can add Line grids and edit inherited Line fields, aggregates, actions, label, visibility, and order; only the current Layer's `lineOverrides` delta is saved. See [Create extensions](../developer/extensions.md) and [Work with metadata layers](../developer/layers.md).

## Review AI proposals

Open **Web Designer → AI Proposals** to inspect pending ChangeSets submitted through the AI REST API. Review the affected Apps, executable Scripts/Functions, diagnostics, and complete diff. Approval revalidates against the current metadata revision; a stale or invalid proposal is not applied. Reject proposals that exceed the intended scope.

## Expected result

Metadata is stored in `designer.db`; additive schema changes are applied to the business database.

## Common errors

- Names are identifiers: use letters, numbers, and underscores without spaces.
- Destructive schema changes and native server code still require a developer.

## Related topics

[Metadata](../developer/metadata.md) · [Views and Charts](../developer/views-and-charts.md) · [Security](../developer/security.md) · [Extensions](../developer/extensions.md) · [Functions and actions](../developer/functions.md) · [Customization checklist](../developer/customization-checklist.md) · [Record lifecycle](../developer/record-lifecycle.md) · [Localization](../developer/localization.md) · [Data Entities](../developer/data-entities.md) · [Model deployment](../developer/model-deployment.md) · [Work with attachments](attachments.md) · [Backup](../admin/backup.md)
