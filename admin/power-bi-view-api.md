# Connect Power BI to the View API

## Purpose

Expose a validated declarative View to Power BI or another HTTPS client without granting access to the generic Data API.

## Prerequisites

- A valid View artifact; see [Build Views and embedded Charts](../developer/views-and-charts.md).
- EmuFramework published behind HTTPS.
- A System Administrator who can create a scoped View token.

## Create a service token

1. Open **Settings → Users & Security**.
2. Under **Power BI / View tokens**, create a token with a descriptive name.
3. Select only the Views the integration needs and optionally set an expiry.
4. Copy the secret immediately. It is shown once; EmuFramework stores only its SHA-256 hash.

The token can call only its named View scopes. It cannot call `/api/data`, Designer, user administration, Chart, or other APIs. Revoke it when the dataset, credential, or integration is retired.

## JSON endpoint

```http
GET /api/views/ERP_CustomerSalesView/data?param.fromDate=2026-01-01&limit=10000&offset=0
Authorization: Bearer emu_view_<secret>
```

JSON defaults to 1,000 rows per page and accepts `limit` from 1 through 10,000 plus a non-negative `offset`. The response includes `schema`, `data`, `limit`, `offset`, and `hasMore`.

Power Query (M) example using a Power BI text parameter named `EmuViewToken`:

```powerquery
let
    Response = Json.Document(
        Web.Contents(
            "https://erp.example.com",
            [
                RelativePath = "api/views/ERP_CustomerSalesView/data",
                Query = [#"param.fromDate" = "2026-01-01", limit = "10000"],
                Headers = [Authorization = "Bearer " & EmuViewToken]
            ]
        )
    ),
    Rows = Table.FromRecords(Response[data])
in
    Rows
```

Configure the Power BI Web data source as anonymous because the M query supplies the HTTPS Bearer header. Keep `EmuViewToken` private and out of source control, reports, logs, and URLs.

## CSV and schema endpoints

```http
GET /api/views/ERP_CustomerSalesView/export?format=csv&param.fromDate=2026-01-01
Authorization: Bearer emu_view_<secret>

GET /api/views/ERP_CustomerSalesView/schema
Authorization: Bearer emu_view_<secret>
```

CSV exports use the server cap configured by `EMU_VIEW_CSV_MAX_ROWS` (100,000 by default). The schema endpoint returns output columns and typed parameters without returning rows.

Never put the token in a query string. EmuFramework accepts it only as `Authorization: Bearer ...` and records creation, last use, revocation, and expiry without logging the secret.

## Related topics

[User and App Access administration](user-security.md) · [Configuration](configuration.md) · [View and Chart metadata](../developer/views-and-charts.md)
