# Deploy models and license ISV add-ons

## Purpose

Move selected Models between environments as a package, and let an independent software vendor (ISV) require an offline license for an add-on Model.

## Audience

ISV developers who ship add-on Models, implementation teams who promote metadata from UAT to production, and administrators who install licenses.

## Prerequisites

Metadata created in Web Designer (file-based Apps keep using their own file deployment), a System Administrator account at each destination, and a checkout of the framework repository for the seller CLI. Selected-model packages and ISV licensing were introduced in v1.3.0. Read [Understand Apps, Models, and Layers](app-model-layer.md) and [Work with metadata layers](layers.md) first.

## Selected-model packages

A selected-model package carries the complete metadata of one or more Models of one App. It replaces the earlier practice of exporting a whole App or a single Model when you want to deploy several Models together.

### Choose a mode

| Mode | Layers allowed | Destination behavior |
| --- | --- | --- |
| `vendor` (vendor update) | `SYS`, `ISV`, `LOC` | Replaces only the selected Models. Keeps the destination's `DEV` and `CUS` metadata and every unselected Model. |
| `promotion` (UAT to Prod) | `SYS`, `ISV`, `LOC`, `DEV`, `CUS` | Replaces each selected Model with its complete source snapshot, including removal of metadata that no longer exists in the source. |

In both modes a selected Model may be empty, which removes all its artifacts but keeps the Model definition. The framework App `system` cannot be deployed.

### Export a package

In Designer, open the App menu and choose **Build deployment package**, or call the endpoint:

```http
POST /api/designer/packages/models/sales/export
Content-Type: application/json

{ "mode": "vendor", "models": ["Core", "Addon"] }
```

The caller needs Customize permission for the App. The response is a package; save it as a JSON file:

```json
{
  "format": "emuframework-metadata",
  "schemaVersion": 2,
  "frameworkVersion": "1.4.0",
  "exportedAt": "2026-10-08T03:15:00.000Z",
  "scope": {
    "type": "models",
    "app": "sales",
    "mode": "vendor",
    "models": [
      { "name": "Core", "label": "Core", "layer": "ISV" },
      { "name": "Addon", "label": "Add-on", "layer": "ISV", "license": { "vendor": "seller-id" } }
    ]
  },
  "artifacts": [],
  "checksum": "<sha256>"
}
```

`artifacts` holds the App manifest plus every artifact of the selected Models, abbreviated above. An empty or missing `models` returns `422` (`Select models to deploy`), as does an unknown Model name. `schemaVersion` 2 is required for `scope.type: "models"`; the earlier App and Model packages remain `schemaVersion` 1 and still import with their original merge behavior. A destination older than 1.3.0 rejects schema version 2. The `checksum` detects corruption. It is not a vendor signature.

### Import a package

1. Upload the file to `POST /api/designer/packages/import/preview` (multipart, 20 MiB limit). The server validates the package and returns a preview with a `previewId`, `expiresAt`, the diff, `schemaEffects`, warnings, the package summary, and `preservedModels` (Models that the package does not touch).
2. Review every creation, change, removal, and schema effect.
3. Confirm with `POST /api/designer/change-sets/apply` and `{ "previewId": "...", "confirmation": true }`. Add `"confirmHighRisk": true` when the preview contains high-risk changes. The preview belongs to the user who created it and expires after 10 minutes.

Deployment is all or nothing. The whole preview fails, and nothing is applied, when:

- a dependency App named in the manifest is not installed (dependencies are never selected automatically; install them or rebuild the package);
- a selected artifact is already owned by a different kind, App, or Model (`Ownership conflict for '<name>'`);
- a Model would change its Layer (`Model '<name>' cannot change layer during deployment`);
- an artifact would replace a file-based artifact;
- an existing extension is no longer compatible with the new base;
- the resulting registry, including local customizations and dependent Apps, no longer validates.

Existing App labels, locale, and other app-level settings at the destination are kept, and required dependencies are merged in. A new App uses the package manifest. The confirm step re-checks the destination revision and returns `409` when metadata changed since the preview.

### Apply semantics

The destination applies the candidate registry, stored metadata, and audit record as one unit. If applying or saving fails, runtime registrations, hooks, events, actions, and database changes made during that apply are restored. Change sets applied from the Designer or AI proposals use the same revalidation and rollback.

Some things stay outside the guarantee:

- Scripts are trusted server code. Static checks cannot prove that an arbitrary `DEV` or `CUS` Script still works with a vendor update, so test vendor updates against realistic customer customizations in UAT.
- A Script that sends email or HTTP requests during deployment cannot be rolled back. Register handlers instead of acting at load time.
- Data tables and columns are never dropped when their metadata disappears. Business data, users, trusted keys, installed licenses, and installation identity are not part of a package.
- The data and designer databases are separate SQLite files. A machine failure between their commits needs recovery from a full backup; take both together before deployment.

## ISV licensing

Licensing is optional and applies only to `ISV` Models that declare a vendor. Models without `license` behave as before.

### Declare the requirement

Add `license` to the Model entry in the App manifest, or set the vendor ID in the Designer's Model dialog:

```json
{
  "kind": "app",
  "name": "sales",
  "label": "Sales",
  "models": [
    { "name": "Core", "label": "Core", "layer": "ISV" },
    { "name": "Addon", "label": "Add-on", "layer": "ISV", "license": { "vendor": "seller-id" } }
  ]
}
```

Only an `ISV` Model can require a license; any other layer fails validation with `Only ISV models may require a license`. Once the requirement exists, ordinary edits and deployments cannot remove it, change the vendor, or move the Model off `ISV`. The API answers `422` (`Cannot remove or change license requirement for '<model>'`), and deleting the Model through customization is refused. Deleting the entire App remains a separate administrative operation. `PUT /api/designer/artifacts/model/:app/:model` accepts `license: { "vendor": "..." }` for the same purpose.

### Seller workflow

Run the CLI from a framework checkout. Generate the key pair once and store the private key where only the seller can read it:

```sh
node scripts/isv-license.mjs keygen /secure/path/seller
```

This writes `/secure/path/seller.private.pem` (Ed25519, PKCS#8) and `/secure/path/seller.public.pem`. The command refuses to overwrite existing files. Never copy the private key to a customer deployment. Key rotation is not supported.

For each customer installation, the customer gives you the customer ID, the installation ID (shown to a System Administrator under **Settings → Apps & Models**), and the App and Model names. Write a payload:

```json
{
  "version": 1,
  "vendor": "seller-id",
  "customer": "customer-id",
  "installationId": "installation-id-from-system-admin",
  "app": "sales",
  "model": "Addon",
  "notBefore": "2026-10-08T00:00:00Z",
  "expiresAt": "2027-10-08T00:00:00Z",
  "sequence": 1
}
```

Then sign it:

```sh
node scripts/isv-license.mjs issue /secure/path/seller.private.pem payload.json customer-license.json
```

The CLI checks that `version` is `1`, that `vendor`, `customer`, `installationId`, `app`, and `model` are non-empty strings, that `sequence` is a positive integer, and that `notBefore` and `expiresAt` are UTC timestamps ending in `Z` with expiry after the start. It will not overwrite the output file.

The output is `{ "payload": {...}, "signature": "<base64>" }`. The signature is Ed25519 over the UTF-8 JSON of the payload with its top-level keys sorted using `localeCompare`. Send the customer the license file and, once, your public key through a channel they trust.

UAT and production are separate installations with separate IDs. Issue one license for each. Never clone a designer database to provision a new environment, because it would clone the installation identity. A full backup restores the same installation; a model package promotes metadata between installations.

### Customer workflow

A System Administrator performs these steps in **Settings → Apps & Models**, which calls the following endpoints (all System Administrator only):

| Endpoint | Body | Purpose |
| --- | --- | --- |
| `PUT /api/system/licenses/customer` | `{ "customer": "customer-id" }` | Register the customer ID once. A different value is refused (`Installation customer is already registered`). |
| `POST /api/system/licenses/vendors` | `{ "vendor": "seller-id", "publicKey": "<PEM>" }` | Trust the seller's Ed25519 key. Verify the key with the seller first. A registered key cannot be replaced. |
| `POST /api/system/licenses/import` | the license file | Install or renew a license. |
| `GET /api/system/apps-models` | none | Inspect Apps, Models, Layers, source (`designer` or `file`), metadata revision, dependencies, deployment history, license status, and the audit trail. |

An import is rejected with `422` unless all of these hold: the vendor is trusted and the signature verifies; the customer and installation IDs match this installation; the App and Model exist, are on the `ISV` Layer, and declare the same vendor; the dates are UTC, `notBefore` is before `expiresAt`, and the license is not already expired; and `sequence` is a positive integer. Every success or rejection is written to the license audit.

### Renewal

Issue a new file with the same identifiers, a later expiry, and a strictly greater `sequence`. A renewal must already be valid when imported (`notBefore` not in the future), so it cannot replace an active license with one that has not started. A first license with a future `notBefore` is accepted but does not permit writes until it starts. The renewal takes effect on the next request, with no redeployment or restart. A page that is already open updates its license notice after a refresh.

### Status and enforcement

| Status | Writes allowed |
| --- | --- |
| `not-required` | Yes, the Model has no `license`. |
| `active` | Yes. |
| `expiring` | Yes. Within 30 days of expiry the system warns users. |
| `missing`, `invalid`, `not-yet-valid`, `expired` | No. There is no grace period. |

When a licensed Model is blocked, its App and every App that depends on it, directly or transitively, run read-only. Metadata stays loaded, and reading, ordinary data exports, backups, and license administration continue to work. The server rejects the following with `403`, code `APP_LICENSE_READ_ONLY`, and the message `App '<app>' is read-only: <app>/<model>: <status>`:

- business writes through the data API and drafts;
- form and line actions and Functions;
- Data Entity import and archive processing;
- attachment changes;
- writes made by hooks, events, and actions registered by a Script of the blocked App, even when another App's request triggers them.

The guard is checked again before a transaction commits, so a license that expires mid-request still prevents the write. An async Function cannot take back an external effect, such as an HTTP call, that it already made before the expiry; any later framework write in that Function is checked again.

`GET /api/metadata` returns `licenseNotices` (`app`, `model`, `status`, `expiresAt`) for Apps the user can open when a license is expiring or blocking.

### Limits

Offline licensing detects tampering with signed payloads. It cannot stop an owner of the server, source, or database from changing the enforcement code, replacing the stored trust, or rolling back the machine clock or a backup. There is no online revocation and no key rotation, and trusted Scripts and native code are not sandboxed. Treat the license as a contractual and operational control, and keep valuable logic out of customer-editable layers.

## Navigation API changes in 1.3.0

Recent navigation became app-local. `GET /api/navigation/preferences` returns `favorites` and `recent` entries shaped `{ app, menuName, itemId, lastOpenedAt }`, with at most ten recent entries per App (the system Settings menu has its own history). `POST /api/navigation/recent` takes `{ menuName, itemId }` and accepts an optional `app`, which must match the menu's owner. The Favorites API and stored data remain for compatibility, although the UI no longer shows stars.

## Procedure

1. Put each add-on in its own `ISV` Model and declare its `license` vendor before the first customer deployment.
2. Build a `vendor` package for updates. Test it in UAT against a copy of the customer's customizations.
3. Generate the key pair, issue a license per customer installation, and record each `sequence`.
4. At the customer, register the customer ID and your public key, import the license, and confirm the status in **Apps & Models**.
5. Rehearse expiry and renewal on a test installation before go-live.

## Related topics

[Apps & Models administration](../admin/apps-models-licenses.md) · [Layers](layers.md) · [Artifact API](artifact-api.md) · [Security](security.md) · [Customization checklist](customization-checklist.md) · [Record lifecycle](record-lifecycle.md)
