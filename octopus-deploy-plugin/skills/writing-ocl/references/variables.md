# Variables OCL

`variables.ocl` is a list of top-level `variable` blocks. **No wrapper** around them.

```ocl
variable "MyVar" { ... }
variable "AnotherVar" { ... }
```

## Variable block structure

```ocl
variable "<VariableName>" {
    type = "<TypeName>"

    value "<inline-string-value>" {
        environment = ["production"]
        channel = ["release"]
        machine = ["web-01"]
        role = ["web-server"]
        action = ["deploy-web"]
        process = ["deployment-process"]
        tenant_tag = ["TagSet/Tag"]

        prompt {
            label = "<UI label>"
            description = "<help text>"
            required = true
        }
    }

    value "<another-value>" { ... }
}
```

The `variable` block label is the variable name (case preserved on first emit, but lookup is case-insensitive).

## Type attribute

The optional `type` attribute appears either on the `variable` block (when all values share the same type — canonical form) or omitted (when the type is `String`, the default).

```ocl
variable "AzureAccount" {
    type = "AzureAccount"

    value "team-ai-test" {
        environment = ["test"]
    }
    value "team-ai-production" {
        environment = ["production"]
    }
}
```

### Supported variable types

| Type | Notes |
|---|---|
| `String` | Default. Plain text, optionally multi-line via heredoc. |
| `Sensitive` | **Cannot be serialized to OCL.** Emit the variable block with type but no value — the user must set the value via UI/API. |
| `Certificate` | Reference to a Certificate stored in the library. Value is a certificate slug. |
| `AzureAccount` | Value is an Azure account slug. |
| `AzureServicePrincipal` | Legacy. Prefer `AzureAccount`. |
| `AzureSubscriptionVariable` | Legacy. Prefer `AzureAccount`. |
| `AzureManagementCertificate` | Legacy. Prefer `AzureAccount`. |
| `AmazonWebServicesAccount` | Value is an AWS account slug. |
| `GoogleCloudAccount` | Value is a GCP account slug. |
| `UsernamePassword` | Value is a `UsernamePasswordAccount` slug. |
| `Token` | Value is a Token account slug. |
| `WorkerPool` | Value is a worker pool slug. |
| `Structured` | JSON / structured value. Limited OCL support. |

For account types, the value's inline label is the **account slug**. The same applies to `Certificate`, `WorkerPool`, etc.

## Value blocks

A variable has **one or more `value` blocks**. Each value defines what the variable evaluates to in a specific scope.

### Inline value (single-line string)

When the value is a short single-line string, put it on the block label:

```ocl
variable "ApiUrl" {
    value "https://api.production.example.com" {
        environment = ["production"]
    }
    value "https://api.staging.example.com" {
        environment = ["staging"]
    }
}
```

### Heredoc / multi-line value

When the value spans multiple lines or is not a string (e.g., JSON), use the `value` attribute inside the block. Omit the block label:

```ocl
variable "Greeting" {
    value {
        value = <<-EOT
            Hello,
            multiline world.
            EOT
    }
}
```

You can mix inline and heredoc forms across multiple `value` blocks of the same variable.

### Empty value (sensitive placeholder)

For sensitive variables, emit the block with no value — the user populates it later:

```ocl
variable "DatabasePassword" {
    type = "Sensitive"
    value {}
}
```

## Scope attributes

A `value` block's scope determines when this value applies. All scope attributes are arrays of slugs (or canonical names for tenant tags). Multiple scope attributes on the same value AND together; multiple slugs within one attribute OR together.

| Attribute | Element format | Meaning |
|---|---|---|
| `environment` | environment slug | When the action runs in any of these environments. Cross-space: `Spaces-N/<slug>`. |
| `channel` | channel slug | When deploying through this channel. |
| `machine` | machine slug | When the action runs on this specific machine. |
| `role` | role slug | When the action targets a machine with this role. |
| `action` | step/action slug | When this specific step's action is running. |
| `process` | process slug | Runbook process scoping. Use `"deployment-process"` for the deployment process or a runbook slug for that runbook. |
| `tenant_tag` | canonical name `TagSet/Tag` | When deploying to a tenant with this tag. |

Example: a value applied only when deploying the `web` step in `production` for a tenant tagged `Region/EU`:

```ocl
variable "RegionalConfig" {
    value "eu-prod-config" {
        action = ["web"]
        environment = ["production"]
        tenant_tag = ["Region/EU"]
    }
}
```

## Prompt block

A `prompt` block makes the variable promptable at deploy/release time:

```ocl
variable "FeatureFlag" {
    value "false" {
        prompt {
            label = "Enable new feature?"
            description = "Set to true to flip the kill-switch."
            required = true
        }
    }
}
```

| Attribute | Type | Notes |
|---|---|---|
| `label` | string | UI label. |
| `description` | string | Help text. |
| `required` | bool | If `true`, deploy/release blocks until provided. |

Only one `prompt` block per `value`.

## Multiple values for the same variable

Define multiple `value` blocks. Most-specific wins (Octopus's resolver picks the value with the most matching scopes):

```ocl
variable "DatabaseConnectionString" {
    value "Server=dev-sql;Database=app;Trusted_Connection=True" {
        environment = ["development"]
    }
    value "Server=prod-sql;Database=app;User Id=#{DBUser};Password=#{DBPassword}" {
        environment = ["production"]
    }
}
```

## Type promotion rule

When all `value` blocks share the same explicit type, the serializer emits a single `type` attribute on the `variable` block (canonical). If types differ across values, each value carries its own `type` (rare).

Authoring guidance: **always put `type` on the `variable` block** when applicable, and let the round-trip handle it if not.

## Sensitive variable handling

- Type `Sensitive` cannot have its value written to OCL — the OCL serializer throws if asked to.
- For sensitive secrets, emit:
  ```ocl
  variable "MySecret" {
      type = "Sensitive"
      value {}
  }
  ```
  Then tell the user: *"Set the value of `MySecret` via the Octopus UI or API. It cannot be stored in Git."*
- Account-type variables (Azure, AWS, GCP) reference accounts by slug. The account itself holds the credentials in Octopus's secret store; the OCL only carries the reference.

## Putting it together — a fuller example

```ocl
variable "AzureAccount" {
    type = "AzureAccount"

    value "team-ai-test" {
        environment = ["test"]
    }
    value "team-ai-production" {
        environment = ["production"]
    }
}

variable "Location" {
    value "eastus2" {}
}

variable "DataZoneCapacity" {
    value "30" {
        environment = ["test"]
    }
    value "300" {
        environment = ["production"]
    }
}

variable "FeatureToggle" {
    value "false" {
        prompt {
            label = "Enable feature toggle?"
            description = "Flips the kill-switch for the new pipeline."
            required = true
        }
    }
}

variable "ReleaseNotes" {
    value {
        value = <<-EOT
            Release for #{Octopus.Project.Name}
              - Built from #{Octopus.Release.Number}
              - Deployed to #{Octopus.Environment.Name}
            EOT
    }
}

variable "ApiKey" {
    type = "Sensitive"
    value {}
}
```
