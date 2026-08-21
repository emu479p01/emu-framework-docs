# Integrate AI through the REST proposal API

## Purpose

Allow an AI client to inspect selected metadata and submit a complete ChangeSet for human review without giving it business-record or apply access.

## Security model

A System Administrator creates a dedicated token under **Settings → Users & Security → AI REST tokens**. Choose one or more existing non-system Apps, the minimum required scopes, and an expiry. The secret is shown once; store it in a secret manager. EmuFramework stores only its SHA-256 hash.

| Scope | Access |
| --- | --- |
| `inspect` | Read capabilities, JSON schemas, and paginated metadata for allowed Apps. |
| `validate` | Validate a ChangeSet and receive diagnostics and a safe diff. |
| `propose` | Validate and place a ChangeSet in the Proposal Inbox. |

Send the token only in the Bearer header:

```http
Authorization: Bearer emu_ai_<secret>
```

Tokens can be expired or revoked and are restricted to their allowed Apps. There is no AI apply endpoint and no AI business-record endpoint.

## Endpoints

```text
GET  /api/v1/ai/capabilities
GET  /api/v1/ai/schemas/artifact
GET  /api/v1/ai/schemas/change-set
GET  /api/v1/ai/workspace?app=&model=&kind=&cursor=&limit=
POST /api/v1/ai/change-sets/validate
POST /api/v1/ai/proposals
```

`workspace` returns `revision`, `artifacts`, `nextCursor`, and `total`. Its default page size is 100 and maximum is 500. Follow `nextCursor` until it is `null`; do not assume the first page is the whole workspace.

## Submit a proposal

1. Call `capabilities` and fetch the Artifact and ChangeSet schemas.
2. Read every required workspace page and keep the returned `revision`.
3. Build a version 1 ChangeSet whose `baseRevision` matches that revision.
4. Validate it and inspect every diagnostic and diff.
5. Submit the same ChangeSet to `proposals`.
6. A customizer opens **Web Designer → AI Proposals**, reviews the complete diff, and approves or rejects it.

```json
{
  "version": 1,
  "baseRevision": "<revision from workspace>",
  "source": "ai",
  "description": "Add the sales order status enum",
  "operations": [
    {
      "op": "upsert",
      "kind": "enum",
      "name": "SALES_OrderStatus",
      "artifact": {
        "kind": "enum",
        "name": "SALES_OrderStatus",
        "app": "sales",
        "model": "Customizations",
        "layer": "CUS",
        "values": [
          { "name": "Open", "value": 0 },
          { "name": "Confirmed", "value": 1 }
        ]
      }
    }
  ]
}
```

Validation returns HTTP `422` for an invalid ChangeSet. Approval revalidates against the current workspace; a stale revision returns a conflict instead of applying outdated metadata. A reviewer must have Customize permission for every affected App.

Scripts and Functions are permitted in proposals because they are reviewed executable Artifacts. Treat them as code: inspect credentials, network access, transaction boundaries, and authorization before approval.

## Auditing and rotation

Token creation, revocation, validation, proposal creation, approval, and rejection are recorded in `designer.db`. Revoke a token immediately if its secret may have leaked; create a replacement rather than trying to recover the old secret.

## Related topics

[Artifact API](artifact-api.md) · [Artifact kinds](artifact-types.md) · [Nested structures](artifact-components.md) · [Metadata](metadata.md) · [Security](security.md) · [Testing](testing.md)
