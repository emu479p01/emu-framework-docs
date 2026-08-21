# Set up a development environment

## Prerequisites

Use Node.js 24.18.0 and pnpm 11.12.0. The declared Node engine range is `^22.13.0 || ^24.0.0`; the pinned toolchain is preferred for reproducible framework work.

## Procedure

1. Clone the repository and run `pnpm install --frozen-lockfile`.
2. Run `pnpm dev` for the API on port 3399 and Vite on port 5199.
3. On a new database, copy the one-time setup code from the API server output and complete **Administrator setup** in the browser. Use a password of at least 12 characters.
4. If the code expires after 15 minutes or ten failed attempts, restart the API server and use the new code.
5. Run `pnpm typecheck`, `pnpm test`, and `pnpm build` before proposing changes.

Development credentials are for local databases only.

Production deployment is Docker-only. The development server is not a supported production installation method.

## Related topics

[Architecture](architecture.md) · [Testing](testing.md)
