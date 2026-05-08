# Process templates OCL

Process templates (also called step templates) are reusable, parameterized pieces of process — a single action or a sequence of steps — that other projects can include. They live in their own `.ocl` files, separate from a specific project's `deployment_process.ocl`.

In OCL, a process template is wrapped in a top-level `process_template { ... }` block. (This is one of the few documents where there *is* a top-level wrapper, because the file may also contain other artifacts.)

## File location

Process templates are stored alongside the project (or in a dedicated process-templates project) under their own `.ocl` files. Use a slug-based file name. The serialization layer is in `source/Octopus.Core/Serialization/Ocl/OclConverters/ProcessTemplateOclConverter.cs`.

## File structure

```ocl
process_template {
    name = "My Process Template"
    description = "Reusable deployment template"

    parameter "my.parameter" {
        display_settings = {
            Octopus.ControlType = "SingleLineText"
        }
        help_text = "Hint shown under the field"
        label = "Field label"
        value "default-value" {}
    }

    parameter "another.parameter" {
        display_settings = {
            Octopus.ControlType = "UsernamePasswordAccount"
        }
        value "some-account-slug" {
            environment = ["Spaces-1/production"]
        }
        value "another-account-slug" {}
    }

    step "step-one" {
        name = "Step One"

        action "hello-world-action" {
            action_type = "Octopus.Script"
            name = "Hello World Action"
            properties = {
                Octopus.Action.Script.ScriptBody = "Write-Host '#{my.parameter}'"
                Octopus.Action.Script.ScriptSource = "Inline"
                Octopus.Action.Script.Syntax = "PowerShell"
            }
        }
    }
}
```

## `process_template` attributes

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | Display name. |
| `description` | string | no | Free text. |

The body contains zero or more `parameter` blocks (in declaration order) followed by one or more `step` blocks (same shape as deployment process steps).

## `parameter` block

```ocl
parameter "<parameter.name>" {
    display_settings = {
        Octopus.ControlType = "<control-type>"
        // additional control-specific keys
    }
    help_text = "..."
    label = "..."

    value "<default-or-environment-specific>" {
        environment = ["..."]
    }
}
```

The block label is the **parameter name** — used inside steps as `#{parameter.name}` for variable substitution.

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `display_settings` | map | yes | At minimum, `Octopus.ControlType` keying the UI control. |
| `help_text` | string | no | Hint text shown in the UI. |
| `label` | string | no | Field label in the UI. Defaults to a humanized form of the name. |

A parameter has zero or more `value` blocks following the same scope/inline/heredoc rules as `variables.ocl` (see `variables.md`). The first unscoped value is the default.

## `Octopus.ControlType` values

The `display_settings.Octopus.ControlType` key tells the UI which control to render and signals the value type. Common values:

| Control type | Use for |
|---|---|
| `SingleLineText` | Plain text value. |
| `MultiLineText` | Multi-line text. |
| `Sensitive` | Secret value (prompt-only; not stored in OCL). |
| `Checkbox` | Boolean. Value `"True"` / `"False"`. |
| `Select` | Dropdown. Requires additional `Octopus.SelectOptions` key with `\|`-separated `key|label` entries. |
| `Certificate` | Reference to a certificate. |
| `AmazonWebServicesAccount` | AWS account picker. |
| `AzureAccount` | Azure account picker. |
| `GoogleCloudAccount` | GCP account picker. |
| `UsernamePasswordAccount` | Username/password account picker. |
| `WorkerPool` | Worker pool picker. |
| `Package` | Package picker. |
| `PackageReferenceList` | Multiple packages. |
| `StepName` | Picker scoped to other steps in the consuming process. |

For `Select`, add the options:

```ocl
display_settings = {
    Octopus.ControlType = "Select"
    Octopus.SelectOptions = "us-east-1|US East (N. Virginia)\nus-west-2|US West (Oregon)\neu-west-1|EU (Ireland)"
}
```

## `value` blocks inside parameters

Same structure as variable values (see `variables.md`):

- Inline string label OR `value = <<-EOT ... EOT` heredoc.
- Optional scope attributes (`environment`, `channel`, `machine`, `role`, `action`, `process`, `tenant_tag`).
- Cross-space environment scoping uses `Spaces-N/<env-slug>` form for templates intended to work across spaces.

```ocl
parameter "deployment.region" {
    display_settings = {
        Octopus.ControlType = "Select"
        Octopus.SelectOptions = "us-east-1|US East\neu-west-1|EU"
    }
    label = "Region"
    value "us-east-1" {}
    value "eu-west-1" {
        environment = ["Spaces-1/eu-prod"]
    }
}
```

## Steps and actions inside a process template

Same as `deployment_process.ocl` — see `deployment-process.md`. Inside the template's steps, reference parameters with the standard `#{parameter.name}` syntax.

```ocl
step "deploy" {
    name = "Deploy"

    action {
        action_type = "Octopus.Package"
        properties = {
            Octopus.Action.Package.FeedId = "octopus-server-built-in"
            Octopus.Action.Package.PackageId = "#{package.id}"
        }

        packages "package-ref" {
            acquisition_location = "Server"
            feed = "octopus-server-built-in"
            package_id = "#{package.id}"
        }
    }
}
```

## Consuming a process template from a project

A consuming project's `deployment_process.ocl` references the template via the `Octopus.Action.Template.Id` and `Octopus.Action.Template.Version` properties on an action of `action_type = "Octopus.StepTemplate"` (or the inferred action type the template carries). See `references/action-types.md` for the exact property keys, and the official Octopus docs for the consume side — that lives outside the template's own OCL.

## Cross-space references

Process templates are commonly shared across spaces. Where a parameter value or step reference points to another space's entity, prefix with `Spaces-N/`:

```ocl
value "production-account" {
    environment = ["Spaces-1/production", "Spaces-2/production"]
}
```

Bare slugs (no prefix) are resolved within the template's own space.
