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

Unit tests cover metadata, security, data APIs, CLI, and UI utilities. Server integration tests use Fastify injection and temporary/in-memory databases. Add authorization and failure-path tests for every administrative endpoint.

## Business-logic test cases

For every hook, event, Script, or Function, test valid input, invalid input, rollback after a failed related write, authorized and unauthorized users, update validation, and delete/reference behavior. Post-events must not be expected to cancel completed operations.

For async Functions, test HTTP/email success and rejection, the 15-second default timeout and configured timeout behavior, non-success HTTP status handling, response/content limits, missing SMTP configuration, rejected recipients, and explicit transaction boundaries. Prove that a service failure does not leave a partial database update unless that partial commit is intentional.

## Debugging links

When metadata does not appear, check the artifact kind/name, JSON shape, app/model scope, references, layer, diagnostics, and generated form/list. When a Function fails, check the action name, request arguments, `executionMode`, permissions, integration configuration, record lookup, field types, transaction boundaries, and duplicate registrations.

Release candidates also require a Windows updater exercise and a Docker smoke update that proves the named volume survives container replacement.

## Related topics

[Contributing to the framework](https://github.com/emu479p01/emu-framework/blob/master/CONTRIBUTING.md) · [Documentation repository](https://github.com/emu479p01/emu-framework-docs) · [Dependency upgrades](dependency-upgrades.md)
