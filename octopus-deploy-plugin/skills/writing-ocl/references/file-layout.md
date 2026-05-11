# `.octopus/` file layout

A Config-as-Code project's OCL lives in a `.octopus/` directory at the root of its Git repository. Each file holds a different concern.

## Canonical tree

```
<repo-root>/
└── .octopus/
    ├── schema_version.ocl       # version = 10
    ├── deployment_process.ocl   # top-level `step` blocks
    ├── deployment_settings.ocl  # connectivity_policy, versioning_strategy, release_notes_template
    ├── variables.ocl            # top-level `variable` blocks
    └── runbooks/
        ├── <runbook-slug-1>.ocl
        ├── <runbook-slug-2>.ocl
        └── ...
```

Every file is required for a complete CaC project, except `runbooks/` which is omitted if the project has no runbooks. `variables.ocl` is required even if empty (write an empty file).

## File-by-file reference

### `schema_version.ocl`

A single attribute:

```ocl
version = 10
```

- The current schema version is **10**.
- Older versions (1–9) are migrated forward automatically when the project is opened by a newer Octopus server.
- New projects should always emit `version = 10`.
- This file is one line plus a trailing newline. Nothing else goes in it.

Schema history (for context — not authoring detail):

| Version | Change |
|---:|---|
| 1 | Initial: deployment process imported into Git |
| 2 | Deployment settings imported |
| 3 | Renamed `schema-version.ocl` → `schema_version.ocl` |
| 4 | Names converted to slugs |
| 5 | Text variables imported |
| 6 | Heredoc indentation fix |
| 7 | Kustomize migration |
| 8 | Kubernetes server-side-apply properties |
| 9 | Runbooks imported |
| 10 | Runbook type discriminator |

### `deployment_process.ocl`

Top-level entries are `step` blocks. There is **no enclosing `deployment_process { ... }` wrapper**. The order of step blocks in the file is the order steps execute.

```ocl
step "first-step" { ... }
step "second-step" { ... }
```

See `deployment-process.md` for the full step/action structure.

### `deployment_settings.ocl`

Top-level entries are attributes and blocks describing project-wide deployment behavior. **No wrapper**.

```ocl
release_notes_template = <<-EOT
    Release #{Octopus.Release.Number}
    EOT

connectivity_policy {
    allow_deployments_to_no_targets = false
}

versioning_strategy {
    template = "#{Octopus.Version.LastMajor}.#{Octopus.Version.LastMinor}.#{Octopus.Version.NextPatch}"
}
```

See `deployment-settings.md` for full attribute/block list.

### `variables.ocl`

Top-level entries are `variable` blocks. **No wrapper**.

```ocl
variable "DatabaseConnectionString" { ... }
variable "FeatureFlag" { ... }
```

See `variables.md` for the full structure of variables, including types, scoping, prompts, and heredoc values.

### `runbooks/<runbook-slug>.ocl`

One file per runbook. **The file's basename (without `.ocl`) is the runbook's slug.** Renaming the file renames the runbook.

The runbook file is a **combined file** containing both metadata (name, description, retention, connectivity policy, multi-tenancy mode, tags) and the runbook process (`process { step { action { ... } } }`).

```ocl
name = "Stored Completions Retention"
description = "Periodically purges stored completions older than 30 days"
default_guided_failure_mode = "EnvironmentDefault"
multi_tenancy_mode = "TenantedOrUntenanted"

connectivity_policy {
    allow_deployments_to_no_targets = true
}

run_retention_policy {
    type = "Default"
}

process {
    step "..." { ... }
}
```

See `runbooks.md` for the full structure.

## File order and stability

The Octopus server's serializer emits attributes and blocks in a deterministic order to keep Git diffs stable. When authoring by hand, follow the canonical ordering you see in the `examples/` directory:

- Inside a `step`: `name`, `package_requirement`, `condition`, `start_trigger`, `properties`, then nested `action` blocks.
- Inside an `action`: `action_type`, `name` (if differs from step), `is_disabled`, `is_required`, `notes`, `condition`, `worker_pool`, `worker_pool_variable`, `tenant_tags`, `environments`, `excluded_environments`, `channels`, `properties`, then nested `container`, `packages`, `git_dependencies` blocks.
- Inside `properties = { ... }`: keys sorted alphabetically.
- Inside a `variable`: `type` first, then `value` blocks.
- Inside a `value`: scope attributes (alphabetical), then nested `prompt` block.

Round-tripping through Octopus will normalize any deviation, so don't worry about exact byte-for-byte alignment when authoring — focus on canonical *shape*.

## Where the serialization rules live (for verification)

- Top-level file handlers: `source/Octopus.Core/Serialization/Ocl/Documents/`
  - `DeploymentProcessOclFileHandler.cs`
  - `DeploymentSettingsOclFileHandler.cs`
  - `ProjectVariablesOclFileHandler.cs`
  - `RunbookOclFileHandler.cs`
- Schema migrations: `source/Octopus.Core/Git/Schema/Migrations/`
- Golden fixtures (for round-trip verification): `source/Octopus.Core.IntegrationTests/Core/Persistence/Git/Migrations/Snapshots/Latest/0010/`
