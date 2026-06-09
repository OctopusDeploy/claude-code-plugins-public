# Octopus Deploy Projects

This document details Octopus Deploy projects, deployment processes, steps, and variables.

## Projects

A project is the primary organizational unit for deploying an application, service, or database component.

### Project Components

- **Deployment Process**: Steps that define how to deploy
- **Runbooks**: Operations tasks (separate from deployment)
- **Variables**: Configuration values
- **Channels**: Different release pipelines
- **Triggers**: Automation rules

### Project Groups

Organize related projects together:
- Logical grouping (e.g., "Online Store" → Website, API, Database)
- Dashboard organization
- Permission boundaries

### Project Settings

**Deployment Settings:**
- Default failure mode (guided/unguided)
- Skip steps on same machine
- Exclude unhealthy targets
- Allow deployments during maintenance windows

**Version Control (Config-as-Code):**
- Store deployment process in Git
- Branch-based development
- Code review for process changes

## Deployment Process

The deployment process is the recipe for deploying software, consisting of ordered steps.

### Process Structure

```
Deployment Process
├── Step 1: Remove from Load Balancer
│   └── Action: Run Script
├── Step 2: Deploy Application
│   ├── Action: Stop Service
│   ├── Action: Deploy Package
│   └── Action: Start Service
└── Step 3: Return to Load Balancer
    └── Action: Run Script
```

### Step Execution

**Sequential Steps:**
- Steps execute in order (Step 1 completes before Step 2)
- Ensures dependencies are respected

**Parallel Actions:**
- Actions within a step run in parallel across targets
- Default: Up to 10 targets simultaneously
- Controlled by `Octopus.Action.MaxParallelism`

### Rolling Deployments

Deploy to targets incrementally rather than all at once.

**Window Size:**
- Number of targets to deploy simultaneously
- Window size 1: One target at a time
- Window size 3: Three targets concurrently

**Child Steps:**
- Group multiple actions to execute on each target
- All child steps complete before moving to next target batch

## Steps and Actions

### Step Types

#### Package Deployment Steps

Deploy packaged applications to targets.

**Features:**
- Configuration transforms (Web.config, appsettings.json)
- Variable substitution in files
- IIS configuration
- Windows Service configuration
- Custom installation directory

**Package Sources:**
- Built-in feed
- NuGet feeds
- Docker registries
- GitHub releases
- Maven repositories

#### Script Steps

Execute custom scripts during deployment.

**Supported Languages:**
- PowerShell (.ps1)
- Bash (.sh)
- Python (.py)
- C# (.csx) via dotnet-script
- F# (.fsx)

**Script Sources:**
- Inline (stored in Octopus)
- Package (script inside deployed package)
- Git repository

#### Built-in Step Templates

Pre-configured steps for common scenarios:

| Step | Purpose |
|------|---------|
| Deploy to IIS | Deploy web applications to IIS |
| Deploy Windows Service | Install/update Windows services |
| Deploy Azure Web App | Deploy to Azure App Service |
| Run Terraform | Execute Terraform plans |
| Deploy Kubernetes YAML | Apply Kubernetes manifests |
| Helm Upgrade | Deploy Helm charts |
| Deploy ECS Service | Update AWS ECS services |

#### Manual Intervention

Pause deployment for human approval.

**Configuration:**
- Responsible teams
- Instructions/notes
- Required environments
- Timeout behavior

#### Health Check Step

Run health checks on deployment targets mid-deployment.

### Step Conditions

Control when and where steps execute.

**Run Conditions:**
- Always run
- Only on success
- Only on failure
- Variable expression (custom logic)

**Environment Scoping:**
- Run in specific environments
- Skip in certain environments

**Channel Scoping:**
- Run only in specific channels

**Target Tag Filtering:**
- Target specific machine roles

### Step Templates

Reusable, parameterized step definitions.

**Types:**
- **Custom Step Templates**: Created by your team
- **Community Step Templates**: Shared by Octopus community

**Parameters:**
- Define inputs for the step
- Set default values
- Control types (text, sensitive, select)

**Versioning:**
- Templates are versioned
- Projects reference specific versions
- Update projects when template changes

## Variables

Variables store configuration values that vary between environments, targets, or tenants.

### Variable Types

**Text Variables:**
```
ConnectionString = Server=db.example.com;Database=MyApp
```

**Sensitive Variables:**
- Encrypted at rest
- Masked in logs
- Cannot be retrieved via API

**Account Variables:**
- Reference to AWS, Azure, SSH accounts
- Syntax: `#{Acme.Azure.Account}`

**Certificate Variables:**
- Reference to stored certificates
- Access thumbprint, PFX, private key

### Variable Scoping

Variables can have multiple values scoped to different contexts:

| Scope Type | Example |
|------------|---------|
| Environment | Different DB per environment |
| Target Tag | Role-specific configuration |
| Machine | Machine-specific settings |
| Channel | Beta vs stable config |
| Step | Step-specific values |
| Tenant | Per-customer configuration |
| Process | Deployment vs runbook |

**Scope Specificity:**
When multiple scopes match, the most specific value wins:
```
Machine > Target Tag > Environment > Unscoped
```

### Variable Syntax

**Basic Substitution:**
```
#{VariableName}
```

**Nested Variables:**
```
#{Octopus.Environment.Name}-#{ProjectName}
```

**Conditional Logic:**
```
#{if Octopus.Environment.Name == "Production"}
  ProductionValue
#{else}
  DefaultValue
#{/if}
```

**Iteration:**
```
#{each item in MyCollection}
  #{item.Name}: #{item.Value}
#{/each}
```

### Variable Filters

Transform variable values during substitution:

| Filter | Example | Result |
|--------|---------|--------|
| ToLower | `#{Name \| ToLower}` | lowercase |
| ToUpper | `#{Name \| ToUpper}` | UPPERCASE |
| HtmlEscape | `#{Html \| HtmlEscape}` | HTML-safe |
| JsonEscape | `#{Json \| JsonEscape}` | JSON-safe |
| Markdown | `#{Notes \| Markdown}` | HTML from MD |
| Replace | `#{Path \| Replace "/" "\\"}` | Path conversion |
| Truncate | `#{Desc \| Truncate 50}` | Limited length |

### System Variables

Built-in variables provided by Octopus:

**Release Variables:**
- `Octopus.Release.Number` - Release version
- `Octopus.Release.Id` - Release ID
- `Octopus.Release.Notes` - Release notes
- `Octopus.Release.Created` - Creation timestamp

**Deployment Variables:**
- `Octopus.Deployment.Id` - Deployment ID
- `Octopus.Deployment.Name` - Deployment name
- `Octopus.Deployment.Created` - Deployment timestamp
- `Octopus.Deployment.CreatedBy.Username` - Who deployed

**Environment Variables:**
- `Octopus.Environment.Name` - Environment name
- `Octopus.Environment.Id` - Environment ID

**Machine Variables:**
- `Octopus.Machine.Name` - Target name
- `Octopus.Machine.Id` - Target ID
- `Octopus.Machine.Hostname` - Target hostname
- `Octopus.Machine.Roles` - Target tags

**Project Variables:**
- `Octopus.Project.Name` - Project name
- `Octopus.Project.Id` - Project ID

**Step Variables:**
- `Octopus.Step.Name` - Current step name
- `Octopus.Step.Number` - Step sequence number

**Action Variables:**
- `Octopus.Action.Name` - Action name
- `Octopus.Action.Package.PackageId` - Package ID
- `Octopus.Action.Package.PackageVersion` - Package version

### Library Variable Sets

Shared variable collections across multiple projects.

**Use Cases:**
- Company-wide configuration
- Shared credentials
- Common environment settings

**Configuration:**
- Create at Library → Variable Sets
- Include in projects via Project Variables → Library Variable Sets

### Output Variables

Pass values between steps.

**Setting Output Variables:**

PowerShell:
```powershell
Set-OctopusVariable -name "DatabaseServer" -value "db01.example.com"
```

Bash:
```bash
set_octopusvariable "DatabaseServer" "db01.example.com"
```

**Using Output Variables:**
```
#{Octopus.Action[StepName].Output.DatabaseServer}
```

**Machine-Specific Outputs:**
```
#{Octopus.Action[StepName].Output[MachineName].DatabaseServer}
```

### Prompted Variables

Request user input when creating a deployment.

**Configuration:**
- Mark variable as "Prompt for value"
- Set label and description
- Define control type (text, multi-line, checkbox)
- Set default value

## Configuration Features

### Configuration Transforms

Apply .NET configuration transforms during deployment.

**Automatic Transforms:**
- `Web.Release.config` applied automatically
- `Web.{Environment}.config` for environment-specific

**Custom Transforms:**
- Specify additional transform files
- Supports multiple transforms

### Variable Substitution in Files

Replace `#{Variable}` syntax in any file.

**Target Files:**
- Specify glob patterns
- Supports JSON, XML, YAML, text files

### Structured Configuration Variables

Replace values in JSON, YAML, XML, and Properties files.

**JSON Example:**
Variable `ConnectionStrings:Database` replaces:
```json
{
  "ConnectionStrings": {
    "Database": "#{ConnectionStrings:Database}"
  }
}
```

## Project Recommendations

### Naming
- Clear, descriptive names
- Consistent conventions
- Include component/service identifier

### Organization
- Group related projects
- Use spaces for team isolation
- Keep project count manageable

### Process Design
- Minimize step count
- Use templates for reusability
- Keep processes focused (single responsibility)
- Document complex steps

### Variables
- Avoid hardcoded values
- Use meaningful names
- Scope appropriately
- Use variable sets for sharing
- Mark sensitive values as sensitive
