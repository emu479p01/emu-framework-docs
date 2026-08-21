# Customization checklist and cautions

## Purpose

Verify a customization — built through Web Designer, a reviewed AI proposal, exported metadata, or Functions/Scripts/native TypeScript — before it reaches production.

## Audience

Application developers, customizers, and reviewers preparing a release.

## Prerequisites

The customization is complete in a development or staging environment. See [Work with metadata](metadata.md), [Create extensions](extensions.md), and [Add business logic](business-logic.md).

## Checklist before deploying a customization

- Check the naming prefix, app, model, and layer.
- Check dependencies whenever an object extends across apps.
- Inspect the inherited read-only chain and confirm the saved Extension contains only the intended Layer delta.
- For Form Extensions, inspect every `lineOverrides` delta and test reset-to-inherited behavior.
- Give Form groups/actions/Charts/lines and menu items stable IDs before targeting presentation overrides.
- Confirm Enum and read-only fields are optional; verify read-only values through trusted code and rejected REST/Form edits.
- Preview the change set and read every destructive or high-risk diff.
- Test permissions with a real user account, not only an administrator.
- Test dynamic lookups when the source is empty, when it changes, and when the referenced record is deleted.
- Test a Function with `showOnCreate` when no `recordId` is available.
- For an async Function, test HTTP/email rejection, service timeouts, non-success HTTP status codes, and explicit transaction boundaries.
- Back up `data.db` and `designer.db`.
- If SMTP is configured, back up `.emu-secret.key` or `EMU_SECRET_KEY_PATH` separately.
- Confirm the Docker volume/network names for each deployment's environment.
- Export the app/model package or keep the metadata files in source control.
- Test Function Extensions with and without `next(args)`, and test View/Chart Extensions against lower-layer changes.

See [Back up databases](../admin/backup.md), [Understand database storage](../admin/database-storage.md), and [Operate Docker](../admin/docker-operations.md).

## Cautions

- Deleting metadata does not immediately delete the underlying physical business data; an orphaned table must be purged deliberately.
- Functions and Scripts are executable code — restrict who can edit them and review every change. See [Develop Functions and actions](functions.md) and [Develop Scripts](scripts.md).
- Lookups cap how many results load at once to avoid oversized dropdowns; design a dedicated search/picker for large datasets.
- Avoid reusing an artifact name across apps — the registry treats names as a global identity.

## Related topics

[Metadata](metadata.md) · [Extensions](extensions.md) · [Web Designer](../user/web-designer.md) · [Testing](testing.md) · [Security](security.md)
