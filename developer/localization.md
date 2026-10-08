# Localize metadata with Translations

## Purpose

Show metadata labels in each user's language without changing the stored labels, and declare the language an App is written in.

## Audience

Application developers, ISV developers, and customizers who deliver Apps to users in more than one language.

## Prerequisites

An App with Models and the artifacts you want to translate. Read [Work with metadata](metadata.md) and [Understand Apps, Models, and Layers](app-model-layer.md) first. Translations were introduced in v1.1.0 and the Designer editors, `defaultLocale`, and framework screen texts in v1.2.0.

## What is translated

A Translation replaces **labels** and report text elements. It does not translate business record values, executable Script or Function code, technical identifiers, or raw server error messages. The stored `label` of an artifact always remains the last-resort text, even when it is written in another language.

## App default locale

An App manifest may declare `defaultLocale`, the language its stored labels are written in. Tags are BCP-47 and are canonicalized with `Intl.getCanonicalLocales` (`th-th` becomes `th-TH`). An App without the property uses `en`. The Designer calls the field **Default language**.

```json
{
  "kind": "app",
  "name": "sales",
  "label": "งานขาย",
  "defaultLocale": "th",
  "models": [{ "name": "Core", "label": "Core", "layer": "ISV" }]
}
```

An invalid tag is rejected with an error that names `defaultLocale`.

## Translation artifact

`translation` is a base kind. `locale` and `resources` are required; `resources` maps a resource key to the translated text. Several Translations may exist for one locale, for example one per Model or Layer.

```json
{
  "kind": "translation",
  "name": "SALES_Thai",
  "app": "sales",
  "model": "Core",
  "layer": "ISV",
  "locale": "th",
  "resources": {
    "app.sales.label": "งานขาย",
    "form.SALES_OrderForm.label": "ใบสั่งขาย",
    "table.SALES_Order.field.amount.label": "จำนวนเงิน",
    "form.SALES_OrderForm.action.confirm.label": "ยืนยัน"
  }
}
```

File-based Apps keep Translations in a `translations` directory next to `tables`, `forms`, and the other kind directories. `locale` must match `^[A-Za-z]{2,3}(?:-[A-Za-z0-9]{2,8})*$` and is stored in canonical form.

## Resource keys

The runtime resolver and the Designer's Translation editor build keys with the same functions, so a key that works in one works in the other.

| Target | Resource key |
| --- | --- |
| App | `app.<app>.label` |
| Model | `model.<app>.<model>.label` |
| Menu | `menu.<menu>.label` |
| Menu item | `menu.<menu>.item.<itemId>.label` |
| Table | `table.<table>.label` |
| Table field | `table.<table>.field.<field>.label` |
| Enum | `enum.<enum>.label` |
| Enum value | `enum.<enum>.value.<valueName>.label` |
| Form | `form.<form>.label` |
| Form group | `form.<form>.group.<groupId>.label` |
| Form action | `form.<form>.action.<actionId>.label` |
| Form line grid | `form.<form>.line.<lineId>.label` |
| Line grid action | `form.<form>.line.<lineId>.action.<actionId>.label` |
| Report | `report.<report>.label` |
| Report parameter | `report.<report>.parameter.<field>.label` |
| Report text element | `report.<report>.element.<elementId>.text` |
| Function, View, Chart, Privilege, Duty, Role, Data Entity | `<kind>.<name>.label` |
| Framework screens (reserved) | `ui.*` |

Groups, actions, lines, menu items, and report elements are addressed by their stable `id`. An item without an `id` cannot be translated, so give identifiers to anything you plan to localize.

## How a label is resolved

Resolution is per key, not per Translation. For one key the resolver walks this chain and uses the first locale that has the key. Duplicates are removed case-insensitively.

```text
user locale (exact) → base language of the user locale
→ owner App defaultLocale (exact) → its base language
→ stored label
```

For a user on `en-GB` and an owner App with `defaultLocale` `th-TH`, the chain is `en-GB → en → th-TH → th → stored label`. A partial Translation affects only the keys it contains. The user's saved locale is never rewritten when they open an App that does not support it.

| User locale | App `defaultLocale` | Translations | Result |
| --- | --- | --- | --- |
| `en` | `en` | `en`, `th` | English |
| `en` | `th` | `th` | Thai, because the App default is the next step |
| `th` | `en` | `en`, `th` | Thai |
| `ja` | `th` | `th` | Thai |
| `en` | not set | none | Stored label |

### Layers

Within one locale, several Translations can define the same key. The highest Layer wins in the order `SYS → ISV → LOC → DEV → CUS`. Between Translations on the same Layer, the artifact `name` breaks the tie (the name that sorts last wins). Results never depend on file load order, so a `CUS` Translation can correct a vendor's wording without editing the vendor's artifact.

### Cross-App rule

A Translation can override only keys owned by its own App or by an App it depends on, directly or through other dependencies. Keys that belong to an App it does not depend on are ignored. Declare `dependsOn` when a customization App translates another App's artifacts.

## Framework texts

Keys starting with `ui.` are reserved for the framework. Only Translations owned by the `system` App can define them. The framework ships two read-only system Translations, `FW_UiEn` and `FW_UiTh`, which cover screen titles, buttons, placeholders, and help texts for the shell, home, attachments, System Maintenance, and the Designer chrome. Some long-tail administration and Designer labels are still English.

## Locale in the API

`GET /api/metadata` returns metadata already localized for the signed-in user and adds:

| Property | Meaning |
| --- | --- |
| `locale` | The user's current locale. |
| `availableLocales` | The union of the framework locales and every `apps[].availableLocales` entry in the response. |
| `uiMessages` | Resolved framework `ui.*` texts for the user's locale. |
| `apps[].defaultLocale` | The App's effective default locale (`en` when not declared). |
| `apps[].availableLocales` | The App's default locale plus every locale that translates its artifacts, including Translations from dependent Apps. |

A user changes language with `PATCH /api/me/locale`.

```http
PATCH /api/me/locale
Content-Type: application/json

{ "locale": "th-th" }
```

The server canonicalizes the tag and returns `{ "ok": true, "locale": "th-TH" }`. An invalid tag returns `400` with an error that names `locale`. Reload `/api/metadata` afterward to receive the new texts.

## Diagnostics

```http
GET /api/designer/translations/diagnostics
```

This Designer endpoint returns `{ "diagnostics": [...] }`. Each item has `kind`, `locale`, `key`, and `message`:

- `duplicate`: several Translations define the same key for a locale. The highest Layer wins; the message lists the artifacts.
- `missing-target`: the key no longer matches a live artifact, field, group, action, or other target.

Both are warnings. They never stop metadata from loading. In the Designer's Translation editor an empty cell means "do not override" and the key is dropped from `resources` on save.

## Procedure

1. Set the App's `defaultLocale` to the language of the stored labels.
2. Give stable `id` values to groups, actions, lines, menu items, and report elements.
3. Create a Translation per locale, and per Model or Layer when different teams own the text.
4. Review `/api/designer/translations/diagnostics` and fix duplicates and stale keys.
5. Sign in with a user in each locale and walk menus, forms, actions, and report parameters. Test a locale you do not translate to confirm the fallback.

## Related topics

[Artifact kinds](artifact-types.md) · [Nested metadata structures](artifact-components.md) · [Metadata](metadata.md) · [Work with metadata layers](layers.md) · [Reports](reports.md)
