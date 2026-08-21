# Understand the architecture

## Purpose

Understand how metadata becomes a secured application and where application behavior belongs.

## Audience

Developers and framework maintainers.

## Prerequisites

Basic familiarity with TypeScript, SQLite, and metadata-driven applications.

## Runtime flow

```mermaid
flowchart TD
    A[Web Designer artifact or reviewed AI proposal] --> B[Schema validation]
    B --> C[Metadata registry]
    C --> D[Kernel registration]
    D --> E[Additive SQLite synchronization]
    D --> F[Fastify API]
    D --> G[Vue generated UI]
    F --> H[Authenticated DataContext]
    G --> H
    H --> I[Validation, hooks, events, transaction]
    I --> J[data.db]
    K[Designer artifacts] --> L[designer.db]
    L --> C
```

## Runtime layers

EmuFramework is a pnpm workspace with three packages: `@emu/core` for metadata, SQLite access, schema synchronization, and security; `@emu/server` for Fastify APIs and runtime services; and `@emu/client` for the Vue application and Web Designer.

The kernel resolves metadata with a single-pass dependency pipeline, reuses content-hash results, synchronizes only affected tables, registers business logic, and enforces security. `data.db` stores business and system records; `designer.db` stores browser-created Artifacts, Designer state, AI tokens, proposals, and audit records.

Production execution is Docker-only. Local Node.js commands are for framework development and verification, not for running a production host.

## Design rules

- Route every data operation through the framework data layer.
- Extend applications through metadata, hooks, events, Scripts, Functions, and declared dependencies.
- Do not edit generated database structure manually.
- Treat client visibility as usability, not authorization.

## Related topics

[Metadata](metadata.md) · [Business logic](business-logic.md) · [Extensions](extensions.md)
