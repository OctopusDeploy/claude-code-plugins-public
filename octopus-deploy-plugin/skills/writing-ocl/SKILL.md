---
name: writing-ocl
description: Authoritative guide to writing OCL (Octopus Configuration Language) for Octopus Deploy Config-as-Code projects. Use when authoring or editing any .ocl file under .octopus/ — deployment processes, runbooks, variables, deployment settings, schema_version, or process templates. Covers full OCL syntax (HCL-like), the property key namespace for every action_type, slug rules for entity references (accounts, feeds, environments, worker pools, channels, tenant tags, teams), and the canonical file layout. Trigger on "OCL", "octopus configuration language", ".ocl", "deployment_process.ocl", "runbook.ocl", "variables.ocl", "config-as-code", "CaC", or any request to author Octopus deployment processes/runbooks declaratively.
---

# Writing OCL for Octopus Config-as-Code

This skill is the authoritative reference for writing OCL — the HCL-like declarative configuration language used by Octopus Deploy's Config-as-Code (CaC) feature. Use it whenever authoring or editing any file under a project's `.octopus/` directory.

For *conceptual* knowledge about Octopus Deploy (what a release is, how lifecycles work, etc.), use the `octopus-deploy-knowledge` skill instead. This skill is purely about producing valid, canonical OCL files.

## Mental model

OCL is HCL-like (similar to Terraform). A Config-as-Code project stores its definition as a tree of `.ocl` files under `.octopus/`:

```
.octopus/
├── schema_version.ocl       # version = 10
├── deployment_process.ocl   # top-level `step` blocks
├── deployment_settings.ocl  # connectivity_policy, versioning_strategy, release_notes_template
├── variables.ocl            # top-level `variable` blocks
└── runbooks/
    └── <runbook-slug>.ocl   # combined: metadata + process { step { action { ... } } }
```

Every reference to another Octopus entity (account, feed, environment, worker pool, channel, team, tenant tag, project) is a **slug** (kebab-case identifier) that **must already exist** in the target Octopus space before the CaC project is deployed. OCL does not create those entities.

## Authoring workflow

Follow this decision tree when generating OCL:

1. **Determine which files to create.** Map the user's intent to file types using `references/file-layout.md`. Always emit `schema_version.ocl` with `version = 10` for any new project.

2. **Pick the action type for each step.** Look up the `action_type` (e.g., `Octopus.Script`, `Octopus.AzurePowerShell`, `Octopus.Kubernetes.Kustomize`, `Octopus.Manual`, `Octopus.TerraformApply`) in `references/action-types.md`. That file lists every supported action_type, its property keys, which keys are required, valid values, and defaults. **Never invent property keys.** If a key is not in `action-types.md` for that action_type, do not emit it.

3. **Resolve every entity reference to a slug.** For accounts, feeds, environments, worker pools, channels, tags, teams, and projects, consult `references/entity-references.md` to learn the slug format and any prefixing rules (`global/<team-slug>` for system teams, `Spaces-N/<env-slug>` for cross-space references, `TagSet/Tag` for tenant tags). Tell the user which entities must pre-exist in Octopus.

4. **Write the OCL.** Use `references/ocl-syntax.md` for grammar (heredocs, lists, maps, escapes, comments) and the entity-specific reference file for the document type you're authoring:
   - `references/deployment-process.md` — `step` / `action` / `packages` / `container` / `git_dependencies` blocks
   - `references/runbooks.md` — combined runbook file structure
   - `references/variables.md` — `variable` blocks, types, scoping, prompts
   - `references/deployment-settings.md` — `connectivity_policy`, `versioning_strategy`, `release_notes_template`
   - `references/process-templates.md` — `process_template` and `parameter` blocks

5. **Reuse a canonical example.** Start from a file in `examples/` rather than from scratch — they are derived from real golden fixtures. Edit attributes, don't reshape the file.

## Intended sequence: what to do before writing OCL

OCL describes a project's deployment process and variables, but a project also has prerequisites that live in the Octopus database — not in `.octopus/`. Writing OCL "from scratch" without any Octopus setup is feasible for the *structure* (steps, actions, properties, heredocs) but impossible for *entity references*, which Octopus validates against its database the moment it loads the file.

**The recommended sequence:**

1. **Create the project shell in Octopus first** (UI or API). This sets project-level fields that never appear in OCL: name, slug, project group, **lifecycle**, default channel, Git connection (repo URL, credentials, base path, default branch). The project does not need a deployment process yet.
2. **Verify the entities your OCL will reference already exist** in the target space — accounts, feeds, worker pools, environments, channels, tenant tags, teams, any projects you'll deploy via `Octopus.DeployRelease`. Note their slugs (kebab-case derived from display names: `"My Worker Pool"` → `my-worker-pool`). Slugs vary per instance — `hosted-ubuntu` exists on Octopus Cloud but a self-hosted server typically has `default-worker-pool-1`.
3. **Pull the empty `.octopus/` skeleton** Octopus pushes on first commit, or initialize one yourself.
4. **Write OCL locally** using this skill, referencing the slugs from step 2.
5. **Commit and push.** Octopus validates the OCL on load and flags any unresolved references.

**Always reliable across instances** (safe to use without checking):
- Built-in feed: `octopus-server-built-in`
- System team: `global/octopus-administrators`
- Default channel: `default` (auto-created with every new project)

**Never assume across instances** (always confirm with the user):
- Worker pool slugs
- Account slugs
- Environment slugs (and which environments are bound to the project's lifecycle)
- Feed slugs (other than the built-in)
- Tenant tag canonical names

**When you write OCL for a user, always finish with a "What must exist in Octopus" list** — every distinct slug your OCL references, grouped by entity type. The user uses that list to verify or create those entities before pushing. Variables of account types (e.g., `type = "AzureAccount"` with `value "team-ai-test"`) are validated lazily at deployment time, not at parse time, so an incorrect account slug only surfaces when a deployment runs.

## Top-level pitfalls (read this before writing anything)

- **Slugs only.** Entity references are kebab-case slugs (lowercase, dashes, ≤ 50 chars), not display names with spaces and not database IDs like `Environments-1`. Both IDs and slugs round-trip, but slugs are canonical.
- **System teams are prefixed `global/`.** `Octopus.Action.Manual.ResponsibleTeamIds = "global/octopus-administrators"`. Space-scoped teams use the bare slug.
- **Built-in feed.** The Octopus built-in package feed is `octopus-server-built-in`. Do not invent a name.
- **Sensitive values.** Sensitive variables cannot be serialized to OCL. Emit a `variable` block with the right `type` and scope but **do not set a value** — the user must populate it via the Octopus UI or API.
- **Heredoc indentation.** Use `<<-EOT` (the `-` strips leading indentation common to all lines). The closing `EOT` must align with the content indent. Never put text on the same line as the opening `<<-EOT`.
- **Action `name` may be omitted** if it equals the step's name. Prefer the canonical (omitted) form.
- **`package_requirement`** is `BeforePackageAcquisition` or `AfterPackageAcquisition` — exact case matters.
- **`#{...}` template syntax** inside string values is preserved verbatim — Octopus expands it at deployment time, not at OCL parse time. Slug-resolution skips any string containing `#{`.
- **Schema version 10 is current.** Older projects migrate forward automatically, but new projects should write `version = 10`.
- **No top-level wrapper blocks** in `deployment_process.ocl`, `deployment_settings.ocl`, or `variables.ocl`. The top-level entries are `step`, attribute/block pairs, and `variable` blocks respectively. Runbooks *do* have a `process { ... }` wrapper inside the runbook file, but no `runbook { ... }` wrapper around it.

## Pointer table — task → reference file

| If you're writing… | Read first |
|---|---|
| Any `.ocl` file (syntax) | `references/ocl-syntax.md` |
| Deciding which file goes where | `references/file-layout.md` |
| `deployment_process.ocl` | `references/deployment-process.md` + `references/action-types.md` |
| A runbook in `runbooks/<slug>.ocl` | `references/runbooks.md` + `references/action-types.md` |
| `variables.ocl` | `references/variables.md` |
| `deployment_settings.ocl` | `references/deployment-settings.md` |
| A process template / step template | `references/process-templates.md` |
| Anything that names another Octopus entity | `references/entity-references.md` |
| Looking up properties for a specific `action_type` | `references/action-types.md` |

## Before-you-finish checklist

Run through this every time before reporting OCL as complete:

- [ ] `schema_version.ocl` is present and contains `version = 10`.
- [ ] Every `action_type` you used appears in `references/action-types.md`.
- [ ] Every `Required: yes` property for each `action_type` is present in that action's `properties = { ... }`.
- [ ] No invented property keys — every key in a `properties` dictionary appears in `action-types.md` for that `action_type`.
- [ ] Every entity reference (account, feed, environment, worker_pool, channel, team, tenant tag, project) is a slug in canonical form (kebab-case, lowercase, no spaces). System teams use the `global/` prefix; tenant tags use `TagSet/Tag` canonical form.
- [ ] Heredocs use `<<-EOT ... EOT` with consistent content indentation. The closing `EOT` is on its own line and aligned with the content.
- [ ] No sensitive values are written into OCL.
- [ ] You told the user which Octopus entities (accounts, feeds, environments, etc.) must already exist before deploying this CaC project.

## Examples

The `examples/` directory contains canonical, copy-pasteable OCL files derived from real golden fixtures. Start from these and edit:

- `examples/schema-version.ocl` — minimal, always identical
- `examples/deployment-settings.ocl` — connectivity policy + donor-package versioning
- `examples/deployment-process-script.ocl` — script step on a worker pool
- `examples/deployment-process-package.ocl` — package deploy step with feed and env scoping
- `examples/deployment-process-kustomize.ocl` — Kubernetes Kustomize step with git dependencies
- `examples/runbook-azure-script.ocl` — azure powershell runbook
- `examples/runbook-with-tags.ocl` — runbook scoped to tenant tags
- `examples/variables-scoped.ocl` — variables with environment, channel, prompt, heredoc, account-type
