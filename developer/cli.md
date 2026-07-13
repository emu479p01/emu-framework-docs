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

## Files and version control

Keep each app's manifest and metadata JSON in the structure the CLI scaffolds. Use the CLI to add apps, modules, objects, and Extensions so names and paths stay consistent. Commit artifacts to Git and review the diff before deploying. Use a file-based app for the base solution and a Web Designer layer for customer-specific customization.

If a file-based app is added while the server is already running, reload metadata from the Designer or restart the service — the running process does not pick up new files on its own.

## Related topics

[Metadata](metadata.md) · [Extensions](extensions.md) · [Customization checklist](customization-checklist.md)
