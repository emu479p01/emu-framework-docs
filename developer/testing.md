# Run tests

## Purpose

Verify metadata, generated UI, server behavior, security, deployment, and failure paths before shipping an application or Extension.

## Prerequisites

Use local databases and development credentials only. Complete [Set up a development environment](setup.md) first.

## Full verification

```sh
pnpm check:versions
pnpm typecheck
pnpm test
pnpm build
docker compose config
```

Use Node.js 24.18.0 and pnpm 11.12.0. `pnpm test` runs the release-policy tests and then the core, server, and client suites. Server tests build the Fastify app with temporary databases and call it through injection.

Unit tests cover metadata, security, data APIs, AI proposals, reporting, and UI utilities. Server integration tests use Fastify injection and temporary/in-memory databases. Add authorization and failure-path tests for every administrative endpoint.

For v0.1.1.0, cover the complete authorization matrix: no Role/no App Access, Role only, App only, Customize only, App plus matching Role/Privilege, and System Administrator. Invoke table, Function, Report, View, import/export, and Designer routes directly so hidden UI cannot mask an authorization defect. Verify last-admin protection, password session revocation, and immediate application of Role/App Access changes.

For Views, test joins, typed parameters, literal and parameter filters, grouping/aggregates, invalid metadata, protected sources, injection attempts, row limits, App dependencies, and row scopes. Test session and service-token requests for valid, wrong-scope, expired, and revoked cases. For Charts, cover every type, parameter binding, missing required parameters, unauthorized Views, and responsive cleanup/resize.

For v0.1.2.0–v0.1.4.0, also test 20 MB metadata-package imports, mobile form focus without automatic zoom, desktop accordion state, every App-data package validation failure, atomic replace/delete, partial backup components, restore restart/rollback, and restored login credentials. Verify the layered-customization migration is idempotent, preserves audit records, and never changes business data.

For Extension deltas, test every supported inherited-layer transition, stable Form/Menu IDs, hidden/order/label overrides, field editability, View columns, Chart measures, and Function `next(args)` behavior. Reject duplicate targets, same/lower-layer Extensions, missing dependencies, mandatory Enum/read-only fields, and structural properties that the Extension schema does not allow. Exercise line confirmation dialogs, the two-axis business grid, and mixed Thai/Latin PDF output.

For v0.5.0.0, test 5,000+ mixed Artifacts, filtered pagination, cursor boundaries, ETags, incremental persistence, cache invalidation, and synchronization of only affected tables. Exercise Form Extension `lineOverrides` for inherited fields, aggregates, actions, labels, visibility, ordering, reset-to-inherited, and added Line grids.

For v1.0.2 and later, request `/api/metadata` as users with and without Function, Report, and Privilege grants and assert that unauthorized form and line actions are absent, then call the omitted action directly and expect `403`.

For v1.1.0–v1.4.0, add these cases:

- **Localization:** per-key fallback across user locale, base language, and App `defaultLocale`; Layer precedence and name tie-breaks; cross-App Translations with and without `dependsOn`; reserved `ui.*` keys; invalid tags on `PATCH /api/me/locale` and `defaultLocale`; and `/api/designer/translations/diagnostics`.
- **Data Entities:** XLSX and CSV ZIP round trips, a bad row in the middle of a file (only that document fails, with an error file), preview reuse (`410`), permission denial, extension merge conflicts, and archive then restore including attachments.
- **Attachments:** size and type limits, a renamed non-image sent to `preview`, parent-record permission inheritance, and a Function image upload that is retried with the same `uploadId`.
- **Reports:** every paper size and `Custom`, `layoutVersion` 1 warnings against version 2 errors, asset and attachment images, missing images, and borders.
- **Record lifecycle:** draft create, save, repeated save, expiry, another user's token, and a metadata change between create and save (`409`); a failed save followed by a corrected one; audit aliases in filters, `Query`, Views, and Reports; forged audit values; UTC datetimes with and without an offset; and rejection of `async` hooks, event handlers, and `ctx.tts()` callbacks.
- **Encrypted fields:** mask round trips, rejected filters and Views, key migration when `encrypted` is toggled, and exclusion from export.
- **Deployment and licensing:** selected-model preview in both modes, ownership conflicts, missing dependencies, stale previews, rollback on apply failure, license signature and renewal rules (including an out-of-order `sequence`), the 30-day warning, and read-only behavior at expiry for the App and its dependents.

For AI REST, test every token scope, App restriction, expiry and revocation, pagination, stale revisions, schema failures, executable Artifact proposals, and audit records. Prove that no endpoint applies a proposal or returns business records, and that approval revalidates the complete ChangeSet with the reviewer's current App permissions.

## Business-logic test cases

For every hook, event, Script, or Function, test valid input, invalid input, rollback after a failed related write, authorized and unauthorized users, update validation, and delete/reference behavior. Post-events must not be expected to cancel completed operations.

For async Functions, test HTTP/email success and rejection, the 15-second default timeout and configured timeout behavior, non-success HTTP status handling, response/content limits, missing SMTP configuration, rejected recipients, and explicit transaction boundaries. Prove that a service failure does not leave a partial database update unless that partial commit is intentional.

## Debugging links

When metadata does not appear, check the artifact kind/name, JSON shape, app/model scope, references, layer, diagnostics, and generated form/list. When a Function fails, check the action name, request arguments, `executionMode`, permissions, integration configuration, record lookup, field types, transaction boundaries, and duplicate registrations.

Release candidates require a Docker smoke update and restore that prove the named volume survives container replacement, unhealthy images roll back, and both SQLite databases pass post-restart integrity checks.

## Related topics

[Contributing to the framework](https://github.com/emu479p01/emu-framework/blob/master/CONTRIBUTING.md) · [Documentation repository](https://github.com/emu479p01/emu-framework-docs) · [Record lifecycle](record-lifecycle.md) · [Model deployment](model-deployment.md) · [Dependency upgrades](dependency-upgrades.md)
