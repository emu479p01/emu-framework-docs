# Design paginated Reports

## Purpose

Build PDF reports with validated data sources, page-aware bands, repeating child rows, and Freeform or Tablix layouts.

## Report structure

A Report selects one `dataSource` table, optional parameters, page settings, and bands. Margins are `[top, right, bottom, left]` in points. Page size, units, and layout version are described below.

| Band | Behavior |
| --- | --- |
| Header | Renders on `firstPage`, `everyPage`, or `lastPage`. Space is reserved only on pages where it renders. |
| Detail | Repeats for each main data-source record. |
| Footer | Renders on `firstPage`, `everyPage`, or `lastPage` and stays inside the physical page. |
| Line source | Repeats related child records identified by `table` and `refField` after each main Detail. |

Legacy `pageHeader` and `pageFooter` metadata remains compatible and is normalized to Header/Footer with `everyPage` behavior in the Designer.

## Paper, units, and layout version

`page.size` is `A3`, `A4`, `A5`, `Letter`, `Legal`, or `Custom`, and `page.orientation` is `portrait` or `landscape`. The default is A4 portrait with 40-point margins. Paper sizes in points are:

| Size | Portrait (width × height) |
| --- | --- |
| `A3` | 842 × 1191 |
| `A4` | 595 × 842 |
| `A5` | 420 × 595 |
| `Letter` | 612 × 792 |
| `Legal` | 612 × 1008 |
| `Custom` | `page.width` × `page.height`, both required and positive |

All stored geometry is in points. `designUnit` (`cm`, `in`, or `px`) controls only how the Designer displays and accepts values, so changing it never moves an element. One centimeter is about 28.35 points, one inch is 72 points, and one pixel is 0.75 point.

```json
{
  "layoutVersion": 2,
  "designUnit": "cm",
  "page": { "size": "Custom", "width": 283, "height": 425, "orientation": "portrait", "margins": [20, 20, 20, 20] }
}
```

`layoutVersion` defaults to `1`. A version 1 Report keeps its original behavior: an element or Tablix that overflows the printable area is reported as a **warning**, and the Report still previews and prints (the response carries an `X-Emu-Report-Warnings` header with the count). A Report saved from the current Designer is version 2, where the same problems are **errors**, the Report cannot be saved or rendered, and these are checked:

- Element width must fit inside the printable width (page width minus left and right margins).
- Element height must fit inside its band, and Tablix explicit column widths must not exceed the printable width.
- Header and footer bands together must fit inside the printable height.

In either version, these are always errors: Custom size without positive `width` and `height`, negative margins, band heights, or element geometry, margins that leave no printable area, and an `image` element without a valid source. Use `POST /api/designer/reports/validate` with `{ "artifact": <report> }` to receive the diagnostics, each with `severity` `error` or `warning`, before saving.

## Images and borders

An `image` element embeds a PNG or JPEG from one of two sources:

- `source: "asset"` uses an entry of the Report's own `assets` array (`id`, `name`, `mimeType`, `dataBase64`). Assets are stored inside the Report, so packages stay self-contained.
- `source: "attachment"` reads an image attached to the current record, either by `attachmentIdField` (a field holding an attachment ID) or by `attachmentName` (the newest attachment with that name).

```json
{
  "id": "logo",
  "type": "image",
  "x": 0, "y": 0, "width": 120, "height": 48,
  "image": { "source": "asset", "assetId": "company-logo", "fit": "contain", "horizontalAlign": "left" }
}
```

`fit` is `stretch`, `contain` (default), `cover`, or `original`. If the asset or attachment is missing or its content is not a valid PNG or JPEG, the PDF shows `[Image unavailable: <reason>]` in place of the image instead of failing the Report. An attachment is only matched against the record being rendered. WebP attachments are not rendered in Reports.

`text`, `field`, and `rect` elements accept `style.borderWidth`, `style.borderColor`, and `style.borderStyle` (`solid`, `dashed`, `dotted`, or `none`). A text or field element draws its border around the element box and keeps its text inside it.

## Fields, audit aliases, and encrypted fields

Field elements and Tablix columns can use the audit aliases (`sys_createdBy`, `sys_createdAt`, `sys_modifiedBy`, `sys_modifiedAt`) as well as the Table's own fields. Encrypted fields cannot be rendered or used as a parameter; the registry rejects the Report. Labels, parameter captions, and text elements can be translated; see [Localize metadata with Translations](localization.md).

## Choose a layout

Freeform bands position text, field, image, line, and rectangle elements on a fixed canvas. A Freeform Detail or Line row is atomic: when its designed height does not fit, the whole row moves to the next page.

Tablix bands define columns with field, label, width, alignment, and format plus header/row styles and border settings. Set `headerHeight` and `rowHeight` deliberately; the renderer uses them to plan pages and repeats the Tablix header after a page break.

Switching a band between Freeform and Tablix removes the previous layout's elements or columns. Confirm the warning only after exporting or copying any design that must be retained.

## Procedure

1. Create a Report in Web Designer and select its App, Model, Layer, and source table.
2. Choose page size (including Custom), orientation, display unit, margins, and a default font.
3. Configure Header/Footer display policy and height.
4. Design the main Detail as Freeform or Tablix.
5. Add each Line source, choose the child table and reference field, and design its repeating band.
6. Add typed parameters and verify their bindings.
7. Validate the Report JSON, save it, and generate PDFs with zero, one, and enough records to cross several pages.

## Fonts and pagination checks

`Roboto` and `Noto Sans Thai` are built in. PDF rendering segments mixed Thai and Latin text by grapheme and selects a font with the required glyphs. Installed report fonts can be used as defaults or per-element/Tablix overrides.

Validate a layout version 2 Report on every paper size you support, with images from an asset and an attachment, a missing attachment, and borders. Test first-, middle-, and last-page Header/Footer policies; long Detail and Line sequences; Tablix page breaks; configured row heights; footer position; mixed Thai/Latin output; empty datasets; and both page orientations.

## Security

Grant the named Report through a Privilege and also grant read access to its data source. A visible menu item is not authorization; direct report requests are checked server-side.

## Related topics

[Web Designer](../user/web-designer.md) · [Attachments](attachments.md) · [Localization](localization.md) · [Configuration and fonts](../admin/configuration.md) · [Security](security.md) · [Testing](testing.md)
