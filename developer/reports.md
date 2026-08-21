# Design paginated Reports

## Purpose

Build PDF reports with validated data sources, page-aware bands, repeating child rows, and Freeform or Tablix layouts.

## Report structure

A Report selects one `dataSource` table, optional parameters, page settings, and bands. Page size is `A4` or `Letter`; orientation is portrait or landscape; margins are `[top, right, bottom, left]` in points.

| Band | Behavior |
| --- | --- |
| Header | Renders on `firstPage`, `everyPage`, or `lastPage`. Space is reserved only on pages where it renders. |
| Detail | Repeats for each main data-source record. |
| Footer | Renders on `firstPage`, `everyPage`, or `lastPage` and stays inside the physical page. |
| Line source | Repeats related child records identified by `table` and `refField` after each main Detail. |

Legacy `pageHeader` and `pageFooter` metadata remains compatible and is normalized to Header/Footer with `everyPage` behavior in the Designer.

## Choose a layout

Freeform bands position text, field, image, line, and rectangle elements on a fixed canvas. A Freeform Detail or Line row is atomic: when its designed height does not fit, the whole row moves to the next page.

Tablix bands define columns with field, label, width, alignment, and format plus header/row styles and border settings. Set `headerHeight` and `rowHeight` deliberately; the renderer uses them to plan pages and repeats the Tablix header after a page break.

Switching a band between Freeform and Tablix removes the previous layout's elements or columns. Confirm the warning only after exporting or copying any design that must be retained.

## Procedure

1. Create a Report in Web Designer and select its App, Model, Layer, and source table.
2. Choose page size, orientation, margins, and a default font.
3. Configure Header/Footer display policy and height.
4. Design the main Detail as Freeform or Tablix.
5. Add each Line source, choose the child table and reference field, and design its repeating band.
6. Add typed parameters and verify their bindings.
7. Validate the Report JSON, save it, and generate PDFs with zero, one, and enough records to cross several pages.

## Fonts and pagination checks

`Roboto` and `Noto Sans Thai` are built in. PDF rendering segments mixed Thai and Latin text by grapheme and selects a font with the required glyphs. Installed report fonts can be used as defaults or per-element/Tablix overrides.

Test first-, middle-, and last-page Header/Footer policies; long Detail and Line sequences; Tablix page breaks; configured row heights; footer position; mixed Thai/Latin output; empty datasets; and both page orientations.

## Security

Grant the named Report through a Privilege and also grant read access to its data source. A visible menu item is not authorization; direct report requests are checked server-side.

## Related topics

[Web Designer](../user/web-designer.md) · [Configuration and fonts](../admin/configuration.md) · [Security](security.md) · [Testing](testing.md)
