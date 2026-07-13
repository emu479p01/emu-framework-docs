# Use the CLI

Run `pnpm emu --help` for the current command list.

## Audience

Developers creating source-controlled applications and metadata.

## Prerequisites

A configured repository root with dependencies installed.

```sh
pnpm emu add-app <name>
pnpm emu add-module <app> <module>
pnpm emu add-object <app> <kind> <name>
pnpm emu add-extension <app> <target>
pnpm emu list
```

`add-extension` prompts interactively for the source Model (when the app has more than one) and derives the canonical name `<AppPrefix>_<ModelName>_<BaseName>_Extension`. It only offers targets that the extending Model's layer is allowed to extend — that is, targets at a strictly lower layer — and resolves the app's `dependsOn` chain when the target belongs to another app.

Run commands from the repository root. Review generated files, register business logic where required, and run typecheck after scaffolding.

## Related topics

[Metadata](metadata.md) · [Extensions](extensions.md)
