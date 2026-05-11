# Deployment process OCL

`deployment_process.ocl` is a sequence of top-level `step` blocks. Each step holds one or more `action` blocks. Steps execute in file order.

For the property keys allowed inside each action's `properties = { ... }`, look up the `action_type` in `references/action-types.md`.

## Step block

```ocl
step "<step-slug>" {
    name = "<Display Name>"
    package_requirement = "BeforePackageAcquisition"  // or "AfterPackageAcquisition"
    condition = "Success"                              // Success | Failure | Always | Variable
    start_trigger = "StartAfterPrevious"               // StartAfterPrevious | StartWithPrevious
    properties = {
        Octopus.Action.ConditionVariableExpression = "#{if FoundryEnabled == \"true\"}true#{/if}"
        Octopus.Action.MaxParallelism = "5"
        Octopus.Action.TargetRoles = "web-server"
    }

    action "<action-slug>" {
        // ...
    }
}
```

### Step attributes

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | Display name shown in the Octopus UI. |
| `package_requirement` | enum | no | `BeforePackageAcquisition` or `AfterPackageAcquisition`. Default: `LetOctopusDecide` (omit). |
| `condition` | string | no | When to run the step. Default `Success`. |
| `start_trigger` | string | no | Whether to run after or with the previous step. |
| `properties` | map | no | Step-level properties (see common keys below). |

### Common step `properties` keys

These live on the **step** (parent of the action), not on individual actions:

- `Octopus.Action.TargetRoles` — target machine roles (CSV string for multiple)
- `Octopus.Action.MaxParallelism` — concurrency cap across machines
- `Octopus.Action.ConditionVariableExpression` — variable expression for conditional execution; e.g. `"#{if FoundryEnabled == \"true\"}true#{/if}"`

**Quirk:** these step-level keys are still namespaced `Octopus.Action.X` (not `Octopus.Step.X`). Octopus stores them in the same dictionary internally; the prefix is historical. Do not invent `Octopus.Step.*` variants — they will not be recognized.

If you see one of these keys appear on an action in older OCL, leave it there for backward compatibility, but new steps should hoist them to the step block.

## Action block

```ocl
action "<action-slug>" {
    action_type = "Octopus.Script"
    name = "<Display Name>"           // omit if it equals the step's name
    is_disabled = false
    is_required = false
    notes = "Free-text reviewer notes"
    condition = "Success"
    worker_pool = "<worker-pool-slug>"
    worker_pool_variable = "<variable-name>"     // mutually exclusive with worker_pool
    tenant_tags = ["TagSet/Tag1", "TagSet/Tag2"]
    environments = ["production"]
    excluded_environments = ["development"]
    channels = ["release-channel"]

    properties = {
        // see references/action-types.md for valid keys for this action_type
    }

    container { ... }
    packages "<package-ref>" { ... }
    git_dependencies { ... }
}
```

The action's slug label is **optional** when the step has a single action whose slug matches the step's slug. The canonical (cleanest) form is:

```ocl
step "deploy-web" {
    name = "Deploy Web"

    action {
        action_type = "Octopus.Package"
        // ...
    }
}
```

If a step has multiple actions, give each one a distinct label:

```ocl
step "configure-and-deploy" {
    name = "Configure and Deploy"

    action "configure" {
        action_type = "Octopus.Script"
        // ...
    }
    action "deploy" {
        action_type = "Octopus.Package"
        // ...
    }
}
```

### Action attributes

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `action_type` | string | **yes** | Exact value from the catalog in `references/action-types.md`. |
| `name` | string | no | Defaults to the step's name. Omit when equal. |
| `is_disabled` | bool | no | `true` to skip this action. |
| `is_required` | bool | no | `true` to mark as required (cannot be disabled at deploy-time). |
| `notes` | string | no | Free text shown in the UI. |
| `condition` | string | no | When to run within the step. |
| `worker_pool` | string (slug) | no | Worker pool slug. See `entity-references.md`. |
| `worker_pool_variable` | string | no | Variable name resolving to a worker pool. Mutually exclusive with `worker_pool`. |
| `tenant_tags` | list of strings | no | Tenant-tag canonical names (`TagSet/Tag`). |
| `environments` | list of strings | no | Environment slugs to scope this action to. |
| `excluded_environments` | list of strings | no | Environment slugs to exclude. |
| `channels` | list of strings | no | Channel slugs to scope this action to. |
| `properties` | map | yes (usually) | Action-type-specific properties. |

### Properties dictionary

The `properties` map is where the action_type-specific configuration lives. Look up the action_type in `references/action-types.md` to learn:

- Which keys are valid
- Which are required (often conditionally — e.g., `Octopus.Action.Script.ScriptBody` is required only when `Octopus.Action.Script.ScriptSource = "Inline"`)
- Allowed values for enum-style keys
- Default values
- Whether a key holds sensitive data

All values are stringified — even booleans (`"True"` / `"False"`) and numbers (`"30"`).

```ocl
properties = {
    Octopus.Action.RunOnServer = "True"
    Octopus.Action.Script.ScriptBody = <<-EOT
        echo "Hello"
        EOT
    Octopus.Action.Script.ScriptSource = "Inline"
    Octopus.Action.Script.Syntax = "Bash"
}
```

Keys are sorted alphabetically in canonical OCL.

## Nested blocks of `action`

### `container`

Run the action inside a container image. The container image is pulled from the specified feed.

```ocl
container {
    feed = "docker-hub"
    image = "octopusdeploy/worker-tools:6.4.0-ubuntu.22.04"
}
```

Both attributes are required when the block is present.

### `packages`

Acquire a package as part of the action. The block takes a label — the package reference name (used to refer to the package in scripts and other properties via `#{Octopus.Action.Package[<reference-name>].Version}` etc.):

```ocl
packages "my-app" {
    acquisition_location = "Server"             // Server | ExecutionTarget | NotAcquired
    feed = "octopus-server-built-in"
    package_id = "MyApp.Web"
    properties = {
        SelectionMode = "immediate"
    }
}
```

For a single primary package on a `Octopus.Package` step, the label can be omitted:

```ocl
packages {
    acquisition_location = "Server"
    feed = "octopus-server-built-in"
    package_id = "MyApp.Web"
}
```

For a "Deploy a Release" step (`Octopus.DeployRelease`), the package label *is* the project slug:

```ocl
packages "other-project" {
    acquisition_location = "NotAcquired"
    feed = "octopus-project-feed"
    package_id = "other-project"
}
```

### `git_dependencies`

Pull a Git repository as part of the action (used by Kustomize, Terraform, and similar steps that operate on source files):

```ocl
git_dependencies {
    repository_uri = "https://github.com/example/my-config.git"
    default_branch = "main"
    git_credential_type = "Library"          // Library | Anonymous
    git_credential_id = "git-credentials-1"  // required when type = Library
}
```

When `Octopus.Action.GitRepository.Source = "Project"` (in `properties`), the action uses the project's own CaC git repository instead, and the `git_dependencies` block is omitted.

## Step types — common patterns

### Inline script

```ocl
step "say-hello" {
    name = "Say Hello"

    action {
        action_type = "Octopus.Script"
        properties = {
            Octopus.Action.RunOnServer = "True"
            Octopus.Action.Script.ScriptBody = <<-EOT
                Write-Host "Hello"
                EOT
            Octopus.Action.Script.ScriptSource = "Inline"
            Octopus.Action.Script.Syntax = "PowerShell"
        }
        worker_pool = "hosted-windows"
    }
}
```

### Manual intervention

```ocl
step "approve-release" {
    name = "Approve Release"

    action {
        action_type = "Octopus.Manual"
        properties = {
            Octopus.Action.Manual.BlockConcurrentDeployments = "False"
            Octopus.Action.Manual.Instructions = <<-EOT
                ## Please review and approve
                Confirm the staging deployment looks healthy before promoting.
                EOT
            Octopus.Action.Manual.ResponsibleTeamIds = "global/octopus-administrators"
            Octopus.Action.RunOnServer = "false"
        }
    }
}
```

### Deploy a package

```ocl
step "deploy-web" {
    name = "Deploy Web"

    action {
        action_type = "Octopus.Package"
        properties = {
            Octopus.Action.EnabledFeatures = ""
            Octopus.Action.Package.DownloadOnTentacle = "False"
            Octopus.Action.Package.FeedId = "octopus-server-built-in"
            Octopus.Action.Package.PackageId = "MyApp.Web"
        }

        packages {
            acquisition_location = "Server"
            feed = "octopus-server-built-in"
            package_id = "MyApp.Web"
        }
    }
}
```

### Deploy a release of another project

```ocl
step "promote-other-project" {
    name = "Promote Other Project"

    action {
        action_type = "Octopus.DeployRelease"
        properties = {
            Octopus.Action.DeployRelease.DeploymentCondition = "Always"
            Octopus.Action.DeployRelease.ProjectId = "other-project"
        }
    }
}
```

### Azure-authenticated PowerShell

```ocl
step "azure-script" {
    name = "Run Azure Script"

    action {
        action_type = "Octopus.AzurePowerShell"
        properties = {
            Octopus.Action.Azure.AccountId = "#{AzureAccount}"
            Octopus.Action.Script.ScriptBody = <<-EOT
                Get-AzResourceGroup
                EOT
            Octopus.Action.Script.ScriptSource = "Inline"
            Octopus.Action.Script.Syntax = "PowerShell"
            OctopusUseBundledTooling = "False"
        }
        worker_pool = "hosted-ubuntu"

        container {
            feed = "docker-hub"
            image = "octopusdeploy/worker-tools:6.4.0-ubuntu.22.04"
        }
    }
}
```

### Kubernetes Kustomize from project Git

```ocl
step "deploy-kustomize" {
    name = "Deploy Kustomize"
    package_requirement = "AfterPackageAcquisition"
    properties = {
        Octopus.Action.TargetRoles = "k8s-cluster"
    }

    action {
        action_type = "Octopus.Kubernetes.Kustomize"
        properties = {
            Octopus.Action.GitRepository.Source = "Project"
            Octopus.Action.Kubernetes.DeploymentTimeout = "180"
            Octopus.Action.Kubernetes.Kustomize.OverlayPath = "overlays/#{Octopus.Environment.Name}"
            Octopus.Action.Kubernetes.ResourceStatusCheck = "True"
            Octopus.Action.Kubernetes.ServerSideApply.Enabled = "False"
            Octopus.Action.Kubernetes.ServerSideApply.ForceConflicts = "False"
            Octopus.Action.KubernetesContainers.DeploymentWait = "False"
            Octopus.Action.Script.ScriptSource = "GitRepository"
            Octopus.Action.SubstituteInFiles.TargetFiles = "*.yaml"
        }
    }
}
```

## Conventions

- Step slugs are kebab-case, ≤ 50 chars.
- Action `name` is omitted when it equals the step's `name`.
- For a step with one action, the action label is also omitted.
- `properties` keys are alphabetized.
- Heredocs use `<<-EOT ... EOT`.
- Boolean and integer property values are quoted strings (`"True"`, `"180"`).
- Packages, container, and git_dependencies are sibling blocks of action, NOT entries inside `properties`.
