# Entity references in OCL

Almost every interesting OCL document references other Octopus entities — accounts, feeds, environments, worker pools, channels, teams, tenant tags, projects. This file is the authoritative guide to **how** those references look in OCL.

## Identifier rule

OCL stores entity references as **slugs**. A slug is:

- All lowercase
- Words separated by single dashes
- Up to 50 characters
- No leading or trailing dashes
- Generated from the entity's display name by lowercasing and replacing whitespace with `-`

Examples: `production`, `team-ai-test`, `octopus-server-built-in`, `hosted-ubuntu`.

The conversion code lives at `source/Octopus.Shared/Util/SlugGenerator.cs`. The fallback for empty input is the literal `blue-ring`.

OCL also accepts **database IDs** (`Environments-3`, `Feeds-1001`, etc.) — the parser round-trips both. **Always prefer slugs** when authoring; they're stable and human-readable. The serializer emits slugs canonically.

## Entity reference table

| Entity | Where it appears in OCL | Identifier form | Notes |
|---|---|---|---|
| **Account** | `Octopus.Action.Azure.AccountId` (action property), `Octopus.Account.Id`, `Octopus.Action.AzureAccount.Variable`, variable `value "<slug>"` for typed-account variables | account slug | E.g. `team-ai-production`. Must pre-exist. |
| **Feed** | `feed = ...` (in `container`, `packages`, `git_dependencies` blocks), `Octopus.Action.Package.FeedId` (action property) | feed slug | Built-in: `octopus-server-built-in` (legacy aliases also accepted: `feeds-builtin-releases`, `Octopus Server (built-in)`, `octopus-server-releases-built-in`). |
| **Worker pool** | `worker_pool = ...` (action attribute), variable `value "<slug>"` for `WorkerPool`-type variables | worker pool slug | Common managed pools: `hosted-ubuntu`, `hosted-windows`, `default-worker-pool-1`. |
| **Worker pool variable** | `worker_pool_variable = "<variable-name>"` | variable name (NOT slug) | Mutually exclusive with `worker_pool`. The variable must resolve at deploy time to a worker pool slug. |
| **Environment** | `environments = [...]`, `excluded_environments = [...]` (action attributes); `environment = [...]` (variable scope) | environment slug | Cross-space: `Spaces-N/<env-slug>` (process templates, system contexts). |
| **Channel** | `channels = [...]` (action attribute); `channel = [...]` (variable scope) | channel slug | Channels are **per-project, per-space unique**; the slug is unambiguous within the consuming project. |
| **Tenant tag** | `tenant_tag = [...]` (variable scope); `tenant_tags = [...]` (action attribute); `tags = [...]` (runbook attribute) | canonical name `TagSet/Tag` | NEVER plain slug. Always `TagSet/Tag`, e.g. `Region/EU`, `Owner/Platform Team`. Case-insensitive. |
| **Team** | `Octopus.Action.Manual.ResponsibleTeamIds`, `Octopus.Action.Email.ToTeamIds` / `CCTeamIds` / `BccTeamIds` (action properties) | team slug; **system teams use `global/<slug>`** | CSV-encoded when multiple teams: `"team-a,team-b,global/octopus-administrators"`. |
| **Project** | `Octopus.Action.DeployRelease.ProjectId` (action property); `package_id` for "Deploy a Release" steps | project slug | The `packages` block label is also the project slug for `Octopus.DeployRelease`. |
| **Lifecycle** | not in OCL | — | Lifecycles are project-level metadata, set outside OCL via project create/update. |
| **Tenant** | not directly | — | Tenants are not referenced by name in OCL. Multi-tenancy is controlled via `multi_tenancy_mode` and tenant tags. |
| **Machine / target** | `machine = [...]` (variable scope) | machine slug | Rarely used — most scoping is by role or environment. |
| **Role** | `role = [...]` (variable scope); `target_roles = [...]` (in `connectivity_policy`); `Octopus.Action.TargetRoles` (step property, CSV) | role slug | Roles are free-form labels assigned to deployment targets. |
| **Step / action** | `action = [...]` (variable scope); `step = "<slug>"` (in `versioning_strategy.donor_package`) | step or action slug | References a step/action *within the same project*. |
| **Process** | `process = [...]` (variable scope) | process slug | `"deployment-process"` for the project's deployment process; runbook slugs for runbooks. |
| **Process template** | inside an action that consumes a template, via `Octopus.Action.Template.Id` and `Octopus.Action.Template.Version` properties | template slug + version | Look up the action's properties in `references/action-types.md`. |
| **Git credentials** | `git_credential_id = "<slug>"` in `git_dependencies` block; required when `git_credential_type = "Library"` | git credentials slug | Set `git_credential_type = "Anonymous"` and omit `git_credential_id` for public repos. |
| **Certificate** | variable `value "<slug>"` for `Certificate`-type variables; `Octopus.Action.Certificate.Variable` (action property) | certificate slug | Variable holds a reference; the certificate itself lives in the library. |

## What MUST pre-exist in Octopus

When a CaC project is deployed, every referenced entity above must already exist in the target space. OCL **does not create** them.

Tell users explicitly: *"Before deploying this CaC project, ensure the following exist in the target Octopus space: \<list of slugs by entity type\>"*. This avoids the most common failure mode for CaC adoption.

## Slug examples for common entities

| Entity | Common slug examples |
|---|---|
| Built-in feed | `octopus-server-built-in` |
| Hosted Ubuntu workers | `hosted-ubuntu` |
| Hosted Windows workers | `hosted-windows` |
| Default worker pool | `default-worker-pool-1` |
| System administrators team | `global/octopus-administrators` |
| Default everyone team | `global/everyone` |
| Project group default | `default-project-group` |

## Special cases

### Built-in feed aliases

The Octopus built-in package feed historically had several names. The slug-resolution layer accepts all of these as valid references, but the canonical, current form is **`octopus-server-built-in`**. Always emit that unless a fixture forces otherwise.

```
✓ feed = "octopus-server-built-in"
~ feed = "feeds-builtin-releases"            (legacy, still accepted)
~ feed = "Octopus Server (built-in)"          (very old display name, still accepted)
~ feed = "octopus-server-releases-built-in"   (legacy)
```

### System teams (`global/` prefix)

System-scoped teams (teams that exist at the cluster level, visible to all spaces) are referenced with the `global/` prefix. Space-scoped teams use the bare slug.

```
✓ Octopus.Action.Manual.ResponsibleTeamIds = "global/octopus-administrators"
✓ Octopus.Action.Manual.ResponsibleTeamIds = "my-space-team"
✓ Octopus.Action.Manual.ResponsibleTeamIds = "my-space-team,global/octopus-administrators"
```

The `global/` prefix is mandatory for system teams; without it, the team won't resolve.

### Cross-space references (`Spaces-N/` prefix)

Process templates and system-level OCL sometimes reference entities in another space. Prefix with `Spaces-N/<slug>` where `N` is the destination space ID.

```
environment = ["Spaces-1/production", "Spaces-2/production"]
```

In normal project OCL (deployment_process, runbooks, variables), bare slugs are fine — they resolve within the project's own space.

### Tenant tags — `TagSet/Tag` canonical form

Tenant tags are NEVER referenced by slug. They use the canonical name `TagSetName/TagName`:

```
tenant_tag = ["Region/EU", "Region/US-East"]
tags = ["Owner/Platform", "Lifecycle/GA"]
```

Both the tag-set name and the tag name are the **display names** — case-insensitive but case-preserving on first emit. Allowed in display names: spaces, alphanumerics, common punctuation. The `/` separator is exact.

### `#{...}` substitutions are preserved verbatim

When a reference contains an Octopus variable substitution, the slug-resolution layer skips it and emits the raw string. The variable is expanded at deployment time.

```
worker_pool = "#{WorkerPool}"
properties = {
    Octopus.Action.Azure.AccountId = "#{AzureAccount}"
}
```

## Verification: where the rules live in code

- Slug↔ID resolution: `source/Octopus.Core/Serialization/Ocl/TinyTypeConversion/TinyTypeConversionStrategyProvider.cs`
- Slug generation rules: `source/Octopus.Shared/Util/SlugGenerator.cs`
- Action property → entity-type map: `source/Octopus.Core/Features/DeploymentProcesses/SpecialVariableSchema.cs`
- Tag canonical name handling: `source/Octopus.Server.MessageContracts/Features/TagSets/TagCanonicalIdOrName.cs`
- Team `global/` prefix handling: search for `OclFriendlySlug` in `source/Octopus.Core/`.
- Built-in feed aliases: search for `feeds-builtin-releases` in `TinyTypeConversionStrategyProvider.cs`.

## Author's checklist

When you write any OCL with entity references:

- [ ] Every reference is a slug (kebab-case, lowercase) — not a display name with spaces, not a database ID like `Environments-1`.
- [ ] Tenant tags use `TagSet/Tag` canonical form, not slugs.
- [ ] System teams use the `global/` prefix.
- [ ] Cross-space references (process templates) use the `Spaces-N/` prefix.
- [ ] Variable substitutions (`#{...}`) are passed through verbatim.
- [ ] You have listed every Octopus entity that must pre-exist in the user's instructions.
