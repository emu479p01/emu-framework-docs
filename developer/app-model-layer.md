# Understand apps, models, and layers

## Purpose

Understand how an EmuFramework application is organized before creating metadata or customizations.

## Audience

Application developers, ISV developers, customizers, and framework administrators.

## Prerequisites

Basic familiarity with [metadata](metadata.md) and the [application workflow](application-workflow.md).

## The relationship

An **App** is the top-level application boundary. An App contains one or more **Models**, and each Model groups a coherent set of metadata definitions. A **Layer** determines the ownership and precedence of those definitions when the same logical artifact is contributed by multiple sources.

```mermaid
flowchart TD
    A[App] --> B[Model]
    B --> C[Layer]
    C --> D[Metadata artifacts]
    D --> E[Tables, forms, menus, security, reports]
    D --> F[Scripts and Functions]
    A --> G[App dependencies]
    G --> A
```

The practical model is:

```text
App
└── Model
    └── Layer
        └── Metadata artifacts
```

Every new business artifact identifies its owning `app` and `model`; its `layer` must agree with the selected Model. The App establishes runtime access, navigation scope, and dependencies. The Model organizes development definitions. The Layer resolves ownership and precedence.

## App

The App manifest defines the application identity, display information, dependencies, and its Models.

```json
{
  "kind": "app",
  "name": "sales",
  "label": "Sales",
  "icon": "grid",
  "dependsOn": [],
  "models": []
}
```

Every App created in v0.1.1.0 starts with zero Models. This includes names such as `erp`, `erp.credit`, and `web`; no App name creates `MiniERPApplication` or another hidden default. Add a Model explicitly before creating an artifact.

Use the App boundary to decide what belongs together, what the application depends on, which users may open or customize it, and how its navigation is filtered. `dependsOn` determines load order and whether a cross-App reference, View, Chart, or Extension is allowed (see [Extensions](extensions.md)).

## Model

A Model is a named grouping within an App. Use Models to separate coherent areas of an application, such as core sales definitions, reporting definitions, or a customer-specific customization set. A Model can carry its own layer assignment in the App manifest, while individual artifacts may also specify ownership and layer according to the metadata contract.

A Model is not a security boundary. Creating or selecting one never grants App entry, Designer access, or business-object permission.

Keep references within the correct App and Model scope. Create the Model and its foundational definitions before forms, menus, security artifacts, Scripts, or Functions that depend on them.

## Layer

Layers describe where a Model or artifact comes from and which definition takes precedence:

```text
SYS < ISV < LOC < DEV < CUS
```

| Layer | Typical owner | Typical use |
| --- | --- | --- |
| `SYS` | Framework | Core system definitions |
| `ISV` | Vendor or product team | Reusable product definitions |
| `LOC` | Localization or site team | Local or regional changes |
| `DEV` | Development team | Development customizations |
| `CUS` | Customer or implementation team | Customer-specific customizations |

Base artifacts at a higher layer override lower-layer artifacts with the same logical identity. Extensions accumulate into their target instead of replacing it. See [Work with metadata layers](layers.md) for the detailed precedence rules.

## Resolution flow

```mermaid
flowchart LR
    A[App manifest] --> B[Models]
    B --> C[Model layer]
    C --> D[Artifact app/model/layer scope]
    D --> E[Dependency and reference validation]
    E --> F[Layer resolution]
    F --> G[Effective application metadata]
    G --> H[Generated UI and API]
```

## Recommended design

1. Define the App and its dependencies; expect `models: []` initially.
2. Add Models that represent coherent application areas and choose each Layer explicitly.
3. Assign the appropriate Model layer and ownership.
4. Add enums, tables, and references.
5. Add forms, menus, reports, and security.
6. Add Hooks, Scripts, and Functions in the same App/Model scope.
7. Use Extensions for additive changes to an existing App/Model.
8. Validate the effective result at runtime and test each relevant layer.

## Common mistakes

- Expecting the App name to create a default Model.
- Attempting to create an artifact before adding and selecting a Model.
- Putting a customization in a product layer instead of `LOC`, `DEV`, or `CUS`.
- Referencing an artifact from another App or Model without declaring the dependency.
- Replacing a base artifact when an Extension should have been used.
- Assuming a Model or Layer automatically grants security access; permissions still require explicit security metadata.
- Treating Framework/System Models as an extension target; they are read-only inspection metadata.

## Related topics

[Application workflow](application-workflow.md) · [Metadata](metadata.md) · [Work with metadata layers](layers.md) · [Extensions](extensions.md) · [Security](security.md)
