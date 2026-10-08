# Set up a development environment

## Prerequisites

Use Node.js 24.18.0 and pnpm 11.12.0, the toolchain pinned by EmuFramework v1.4.0. The declared Node engine range is `^22.13.0 || ^24.0.0`; the pinned toolchain is preferred for reproducible framework work, and the automated test suites are run on Node 24.

## Procedure

1. Clone the repository and run `pnpm install --frozen-lockfile`.
2. Run `pnpm dev` for the API on port 3399 and Vite on port 5199.
3. On a new database, copy the one-time setup code from the API server output and complete **Administrator setup** in the browser. Use a password of at least 12 characters.
4. If the code expires after 15 minutes or ten failed attempts, restart the API server and use the new code.
5. Run the verification commands below before proposing changes.

## Verify your change

```sh
pnpm check:versions
pnpm typecheck
pnpm test
pnpm build
```

`pnpm check:versions` confirms that the workspace package versions agree with the release being prepared. `pnpm test` runs the release-policy tests and then the core, server, and client suites. The `pnpm check:release -- --tag <X.Y.Z> --previous <X.Y.Z>` command validates release notes and version numbers when you prepare a release. Releases use a three-component `Major.Minor.Patch` version: a Framework Update (FU) raises `Major` or `Minor`, and a Proactive Update (PU) raises `Patch`.

## Environment variables for development

| Variable | Purpose |
| --- | --- |
| `EMU_DB_PATH`, `EMU_DESIGNER_DB_PATH` | Locations of the data and designer SQLite databases. |
| `EMU_FILE_STORAGE_PATH` | Attachment storage. Without it, development uses a per-process temporary directory. |
| `EMU_ARCHIVE_STORAGE_PATH` | Archive storage, with the same development default. |
| `EMU_SECRET_KEY_PATH` | Location of the encryption key. By default the key sits beside the data database. |
| `EMU_APP_TITLE` | Product name shown in the browser title and on the login and setup pages. |

Development credentials are for local databases only.

Production deployment is Docker-only. The development server is not a supported production installation method.

## Related topics

[Architecture](architecture.md) · [Testing](testing.md) · [Development overview](development-guide.md)
