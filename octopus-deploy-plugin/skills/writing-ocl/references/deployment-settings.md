# Deployment settings OCL

`deployment_settings.ocl` holds project-wide deployment configuration. Top-level entries are attributes and blocks. **No wrapper.**

A minimal valid file is just:

```ocl
connectivity_policy {
    allow_deployments_to_no_targets = true
}
```

A more typical file:

```ocl
release_notes_template = <<-EOT
    Release notes for #{Octopus.Project.Name} #{Octopus.Release.Number}
    EOT

connectivity_policy {
    allow_deployments_to_no_targets = false
    exclude_unhealthy_targets = true
    skip_machine_behavior = "SkipUnavailableMachines"
    target_roles = ["web-server", "app-server"]
}

versioning_strategy {
    template = "#{Octopus.Version.LastMajor}.#{Octopus.Version.LastMinor}.#{Octopus.Version.NextPatch}"
}
```

## Top-level attributes

| Attribute | Type | Notes |
|---|---|---|
| `release_notes_template` | string (heredoc) | Template applied when generating release notes. Octopus variable substitution (`#{...}`) supported. |

## Top-level blocks

### `connectivity_policy`

Project-level connectivity rules. Same block shape as in runbooks.

```ocl
connectivity_policy {
    allow_deployments_to_no_targets = false
    exclude_unhealthy_targets = false
    skip_machine_behavior = "None"
    target_roles = ["web-server"]
}
```

| Attribute | Type | Default | Notes |
|---|---|---|---|
| `allow_deployments_to_no_targets` | bool | `false` | Allow a deployment to succeed even if no machines match the target roles. Common for "infrastructure" projects that run on the Octopus server. |
| `exclude_unhealthy_targets` | bool | `false` | Skip targets reporting Unhealthy at deploy time. |
| `skip_machine_behavior` | enum | `"None"` | `None` \| `SkipUnavailableMachines`. |
| `target_roles` | list of role slugs | `[]` | Restrict deployments to machines in these roles. |

### `versioning_strategy`

Determines how new releases are versioned. Two mutually exclusive forms:

#### Template-based versioning

```ocl
versioning_strategy {
    template = "#{Octopus.Version.LastMajor}.#{Octopus.Version.LastMinor}.#{Octopus.Version.NextPatch}"
}
```

The `template` is a string with Octopus version-template variables. Common ones:

- `#{Octopus.Version.LastMajor}` / `LastMinor` / `LastPatch` — values from the previous release
- `#{Octopus.Version.NextPatch}` / `NextMinor` / `NextMajor` — auto-increment relative to last release
- `#{Octopus.Action[<step-slug>].Output.Package.Version}` — derive from a package step's output

#### Donor-package versioning

```ocl
versioning_strategy {
    donor_package {
        step = "<step-slug>"
        package = "<package-reference>"   // optional — required only when the step has multiple packages
    }
}
```

The release version is taken from the package version of the named step. If the step has only one package, omit `package`.

### `release_creation_strategy` (less common)

Defines how new releases are created from triggers. When omitted, releases are created manually.

```ocl
release_creation_strategy {
    channel = "release"                       // optional
    release_creation_package_step = "<slug>"  // step whose package version triggers releases
    release_creation_package {
        deployment_action = "<action-slug>"
        package_reference = "<package-ref>"
    }
}
```

This block appears rarely. Most projects rely on triggers configured at the space level rather than in the project's CaC.

## Common patterns

### Server-only project (no targets)

```ocl
connectivity_policy {
    allow_deployments_to_no_targets = true
}

versioning_strategy {
    template = "#{Octopus.Version.LastMajor}.#{Octopus.Version.LastMinor}.#{Octopus.Version.NextPatch}"
}
```

This is the canonical form for projects whose steps all run on workers (Terraform, scripts) rather than against deployment targets.

### Web-app project

```ocl
release_notes_template = <<-EOT
    Web release #{Octopus.Release.Number} from #{Octopus.Project.Name}
    EOT

connectivity_policy {
    exclude_unhealthy_targets = true
    target_roles = ["web-server"]
}

versioning_strategy {
    donor_package {
        step = "deploy-web"
    }
}
```

The release version is taken from the package built and deployed by the `deploy-web` step.
