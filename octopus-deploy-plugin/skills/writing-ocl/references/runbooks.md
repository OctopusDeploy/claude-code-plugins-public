# Runbooks OCL

A runbook lives in a single file at `.octopus/runbooks/<runbook-slug>.ocl`. **The file's basename is the slug** — renaming the file renames the runbook.

Each runbook file is **combined**: it holds both the runbook's metadata (name, description, retention, multi-tenancy, connectivity policy, tags) and its process (`process { step { action { ... } } }`).

## File structure

```ocl
name = "Stored Completions Retention"
description = "Periodically purges stored completions older than 30 days"
default_guided_failure_mode = "EnvironmentDefault"
multi_tenancy_mode = "TenantedOrUntenanted"
tags = ["TagSet/Maintenance", "TagSet/Scheduled"]

connectivity_policy {
    allow_deployments_to_no_targets = true
}

run_retention_policy {
    type = "Default"
}

process {
    step "run-an-azure-script" {
        name = "Run an Azure Script"

        action {
            action_type = "Octopus.AzurePowerShell"
            properties = {
                Octopus.Action.Azure.AccountId = "#{AutomationAccount}"
                Octopus.Action.Script.ScriptBody = <<-EOT
                    Write-Host "Doing maintenance"
                    EOT
                Octopus.Action.Script.ScriptSource = "Inline"
                Octopus.Action.Script.Syntax = "PowerShell"
            }
            worker_pool = "hosted-windows"
        }
    }
}
```

There is **no enclosing `runbook { ... }` wrapper** at the top of the file. The top-level entries are attributes and blocks.

## Top-level attributes

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `name` | string | **yes** | Display name. |
| `description` | string | no | Free text. Use a heredoc for multi-line. |
| `default_guided_failure_mode` | enum | no | `Off` \| `On` \| `EnvironmentDefault`. Defaults to `EnvironmentDefault`. |
| `multi_tenancy_mode` | enum | no | `Untenanted` \| `Tenanted` \| `TenantedOrUntenanted`. Defaults to `Untenanted`. |
| `tags` | list of canonical tags | no | Each entry is `TagSet/Tag`. See `entity-references.md`. |

`name` is always emitted first by the serializer. Keep it first when authoring.

## Top-level blocks

### `connectivity_policy`

Same shape as in `deployment_settings.ocl`:

```ocl
connectivity_policy {
    allow_deployments_to_no_targets = true
    exclude_unhealthy_targets = false
    skip_machine_behavior = "None"        // None | SkipUnavailableMachines
    target_roles = ["maintenance-target"]
}
```

For runbooks that run inline on the Octopus server (no targets), set `allow_deployments_to_no_targets = true`. This is the most common case.

### `run_retention_policy`

How long to retain runbook run history:

```ocl
run_retention_policy {
    type = "Default"
}
```

Or with a specific count:

```ocl
run_retention_policy {
    type = "Count"
    quantity_to_keep = 86
}
```

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `type` | enum | **yes** | `Default` \| `Count`. |
| `quantity_to_keep` | integer | when `type = "Count"` | Number of runs to keep. |

### `process`

Wraps the runbook's process steps. **This is the only block in OCL where step blocks are nested inside a parent wrapper.** Inside `process { ... }`, the step/action structure is identical to `deployment_process.ocl`. See `deployment-process.md` for details.

```ocl
process {
    step "first-step" { ... }
    step "second-step" { ... }
}
```

## Runbook-only constraints

- Runbooks do not have channels — `channels = [...]` on actions is ignored.
- Runbooks share the project's variable set; do not duplicate variables in the runbook.
- Runbooks share the project's environments; the action-level `environments = [...]` and `excluded_environments = [...]` lists work the same way.
- A runbook's `multi_tenancy_mode` is independent of the project's deployment process. A project can be tenanted while a maintenance runbook is untenanted, or vice versa.

## Tags

Runbook tags are canonical tag names — `TagSet/Tag` — not slugs. Tags must already exist in the target Octopus space. See `entity-references.md` for canonical-name rules.

```ocl
tags = ["Maintenance/Scheduled", "Owner/Platform Team"]
```

## Common runbook patterns

### Scheduled maintenance script

```ocl
name = "Cleanup Old Resources"
description = "Daily cleanup of stale resources"
default_guided_failure_mode = "Off"
multi_tenancy_mode = "Untenanted"

connectivity_policy {
    allow_deployments_to_no_targets = true
}

run_retention_policy {
    type = "Count"
    quantity_to_keep = 30
}

process {
    step "cleanup" {
        name = "Cleanup"

        action {
            action_type = "Octopus.AzurePowerShell"
            properties = {
                Octopus.Action.Azure.AccountId = "#{AutomationAccount}"
                Octopus.Action.Script.ScriptBody = <<-EOT
                    # cleanup logic
                    EOT
                Octopus.Action.Script.ScriptSource = "Inline"
                Octopus.Action.Script.Syntax = "PowerShell"
            }
            worker_pool = "hosted-windows"
        }
    }
}
```

### Multi-step Terraform plan + manual + apply

```ocl
name = "Apply Infrastructure Changes"
description = "Plans and applies Terraform changes after manual review"
default_guided_failure_mode = "EnvironmentDefault"
multi_tenancy_mode = "Untenanted"

connectivity_policy {
    allow_deployments_to_no_targets = true
}

run_retention_policy {
    type = "Default"
}

process {
    step "tf-plan" {
        name = "Tf Plan"

        action {
            action_type = "Octopus.TerraformPlan"
            properties = {
                Octopus.Action.AzureAccount.Variable = "AzureAccount"
                Octopus.Action.GitRepository.Source = "Project"
                Octopus.Action.Script.ScriptSource = "GitRepository"
                Octopus.Action.Terraform.AzureAccount = "True"
                Octopus.Action.Terraform.ManagedAccount = "None"
                Octopus.Action.Terraform.PlanJsonOutput = "False"
                Octopus.Action.Terraform.RunAutomaticFileSubstitution = "True"
                Octopus.Action.Terraform.TemplateDirectory = "infrastructure"
                OctopusUseBundledTooling = "False"
            }
            worker_pool = "hosted-ubuntu"

            container {
                feed = "docker"
                image = "octopusdeploy/worker-tools:6.4.0-ubuntu.22.04"
            }
        }
    }

    step "confirm-plan" {
        name = "Confirm Plan"

        action {
            action_type = "Octopus.Manual"
            properties = {
                Octopus.Action.Manual.BlockConcurrentDeployments = "False"
                Octopus.Action.Manual.Instructions = "Review the plan output and approve to apply."
                Octopus.Action.RunOnServer = "false"
            }
        }
    }

    step "tf-apply" {
        name = "Tf Apply"

        action {
            action_type = "Octopus.TerraformApply"
            // same properties as tf-plan, plus apply-specific ones
        }
    }
}
```
