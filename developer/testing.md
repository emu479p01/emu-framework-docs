# Run tests

## Full verification

```sh
pnpm check:versions
pnpm typecheck
pnpm test
pnpm build
docker compose config
```

Unit tests cover metadata, security, data APIs, CLI, and UI utilities. Server integration tests use Fastify injection and temporary/in-memory databases. Add authorization and failure-path tests for every administrative endpoint.

Release candidates also require a Windows updater exercise and a Docker smoke update that proves the named volume survives container replacement.

## Related topics

[Contributing to the framework](https://github.com/emu479p01/emu-framework/blob/master/CONTRIBUTING.md) · [Documentation repository](https://github.com/emu479p01/emu-framework-docs) · [Dependency upgrades](dependency-upgrades.md)
