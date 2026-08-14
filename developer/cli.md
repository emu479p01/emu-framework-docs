# Use the CLI

Run `pnpm emu --help` for the current command list.

## Audience

Developers creating source-controlled applications and metadata.

## Prerequisites

A configured repository root with dependencies installed.

```sh
pnpm emu add app <name>
pnpm emu add model <app> <name> --layer <SYS|ISV|LOC|DEV|CUS>
pnpm emu add module <app> <module>
pnpm emu add object <app> <kind> [module] --model <model> --name <name>
pnpm emu add extension <app> <target>
pnpm emu list
```

`add app` creates an App manifest with `models: []`; it never infers a Model from the App name. Run `add model` before `add object`. `add object` rejects an App with no Models and requires `--model` in non-interactive use; interactive use asks the user to select an existing Model.

`add extension` prompts for `tableExtension`, `formExtension`, `menuExtension`, `viewExtension`, `chartExtension`, or `functionExtension`, then for the source Model when needed. It derives the canonical name `<AppPrefix>_<ModelName>_<BaseName>_Extension`, only offers targets at a strictly lower Layer, and resolves the App's `dependsOn` chain when the target belongs to another App. Other supported Extension kinds can be authored through Designer or validated JSON.

Run commands from the repository root. Review generated files, register business logic where required, and run typecheck after scaffolding.

## Files and version control

Keep each App's manifest and metadata JSON in the structure the CLI scaffolds. Use the CLI to add Apps, Models, modules, objects, and Extensions so names and paths stay consistent. Commit artifacts to Git and review the diff before deploying. Use a file-based App for the base solution and a Web Designer layer for customer-specific customization.

If a file-based App is added while the server is already running, reload metadata from Designer or restart the service — the running process does not pick up new files on its own.

## Related topics

[Metadata](metadata.md) · [Extensions](extensions.md) · [Customization checklist](customization-checklist.md)
