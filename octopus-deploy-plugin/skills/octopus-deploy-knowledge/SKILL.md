---
name: octopus-deploy-knowledge
description: Comprehensive knowledge about Octopus Deploy, including core concepts, deployment processes, variables, releases, lifecycles, tenants, runbooks, AI-powered features, and best practices. Trigger when user mentions Octopus Deploy or general software deployment concepts.
---

# Octopus Deploy Knowledge Skill

This skill provides comprehensive knowledge about Octopus Deploy, a deployment automation platform that enables continuous delivery for complex application deployments. Use this knowledge when helping users understand Octopus Deploy concepts, troubleshoot deployments, or design deployment pipelines.

## What is Octopus Deploy?

Octopus Deploy is a deployment automation server that takes packages and artifacts from build servers and deploys them to various targets (Windows, Linux, Azure, AWS, Kubernetes, and more) using safe, consistent, and repeatable processes. It sits between your CI/CD build server and your deployment targets, providing:

- **Release Management**: Snapshot and version your deployment processes
- **Environment Progression**: Promote releases through Dev → Test → Staging → Production
- **Configuration Management**: Manage environment-specific variables and secrets
- **Infrastructure Orchestration**: Deploy to diverse targets from a single platform
- **Operations Automation**: Run day-2 operations tasks via Runbooks
- **Multi-Tenancy**: Deploy to multiple customers/regions with tenant-specific configurations

## Core Concepts and Their Relationships

### The Deployment Pipeline Flow

```
Package Repository → Project → Release → Deployment → Environment → Deployment Targets
                         ↓
                   Deployment Process
                   (Steps with Variables)
```

### 1. Octopus Server

The central orchestration hub responsible for:
- Hosting the Web Portal and REST API
- Storing configuration in SQL Server database
- Orchestrating deployments and runbooks
- Managing users, teams, and permissions

**Deployment Options:**
- **Self-Hosted**: Install on your infrastructure; you control upgrades and maintenance
- **Octopus Cloud**: Hosted by Octopus Deploy; they manage upgrades and infrastructure

### 2. Spaces

Spaces partition an Octopus Server for multi-team isolation. Each space has completely separate:
- Projects and Project Groups
- Environments and Deployment Targets
- Lifecycles and Channels
- Variables and Variable Sets
- Tenants and Tenant Tags
- Step Templates

**Key Point**: Resources in Space-A are NOT accessible to projects in Space-B. Use the Default Space if you don't need isolation.

### 3. Projects

A Project is the primary container for deploying an application, service, or database. Each project contains:
- **Deployment Process**: The steps to deploy your software
- **Variables**: Configuration values (connection strings, API keys, feature flags)
- **Runbooks**: Operations tasks (backups, restarts, provisioning)
- **Channels**: Different release pipelines (stable, beta, hotfix)
- **Triggers**: Automatic release creation and deployment

**Project Groups**: Organize related projects together (e.g., "Online Store" containing Website, API, Database projects).

### 4. Environments

Environments represent stages in your deployment pipeline where software is deployed:
- **Development**: For developer experimentation
- **Test/QA**: Quality assurance testing
- **Staging/Pre-Production**: Final validation before production
- **Production**: End-user facing environment

**Key Points**:
- Environments are ordered (least production-like first)
- Deployment targets are assigned to environments
- Variables can be scoped to specific environments
- Permissions can restrict who deploys to each environment

### 5. Deployment Targets

Physical or virtual destinations where software is deployed:

**Target Types**:
- **Tentacle (Windows/Linux)**: Lightweight agent for Windows and Linux servers
  - *Listening Tentacle*: Octopus connects to Tentacle (TCP server on target)
  - *Polling Tentacle*: Tentacle connects to Octopus (for NAT/firewall scenarios)
- **SSH Targets**: Connect to Linux/Unix machines via SSH
- **Kubernetes Clusters**: Deploy to K8s via Kubernetes Agent or API
- **Azure Web Apps/Cloud Services**: Deploy to Azure PaaS
- **AWS ECS/Lambda**: Deploy to AWS services
- **Offline Package Drop**: Generate packages for manual deployment

**Target Tags (Roles)**: Logical groupings like "web-server", "database-server" used to target steps at specific machines.

### 6. Workers

Machines that execute deployment work that doesn't run on deployment targets:
- Run scripts, execute Terraform, run database migrations
- Offload work from the Octopus Server

**Worker Types**:
- **Built-in Worker**: Runs on Octopus Server (self-hosted only)
- **Dynamic Workers**: On-demand VMs managed by Octopus Cloud
- **External Workers**: Self-managed Tentacles or SSH machines in Worker Pools

### 7. Deployment Process

The recipe for deploying software, consisting of **Steps** that contain **Actions**:

**Step Types**:
- **Package Deployment Steps**: Deploy NuGet, ZIP, Docker images
- **Script Steps**: Run PowerShell, Bash, Python, C#, F#
- **Built-in Steps**: IIS configuration, Windows Services, certificates
- **Step Templates**: Reusable, parameterized steps shared across projects

**Execution**:
- Steps execute sequentially by default
- Actions within a step execute in parallel across targets
- Rolling deployments control the parallelism (window size)
- Conditions control when steps run (environments, channels, variable expressions)

### 8. Variables

Store configuration that varies between environments, targets, or tenants:

**Variable Types**:
- **Text Variables**: Connection strings, URLs, feature flags
- **Sensitive Variables**: Passwords, API keys (encrypted at rest)
- **Account Variables**: AWS, Azure, SSH credentials
- **Certificate Variables**: SSL certificates

**Scoping**: Variables can be scoped to:
- Environments (different DB connection per environment)
- Deployment Targets or Target Tags
- Channels (beta vs stable)
- Steps or Actions
- Tenants

**Variable Sets**: Shared variable collections reusable across multiple projects.

**System Variables**: Built-in variables like `Octopus.Environment.Name`, `Octopus.Release.Number`, `Octopus.Machine.Name`.

### 9. Releases

A **Release** is a snapshot of:
- The deployment process (steps and their configuration)
- Variable values (except tenant variables)
- Package versions

**Key Points**:
- Releases are immutable - changes to the process don't affect existing releases
- Each release has a version number (typically SemVer)
- Releases are deployed to environments
- The same release can be deployed multiple times

### 10. Lifecycles

Control how releases progress through environments:

**Phases**: Ordered stages containing one or more environments
- Define which environments are available
- Control automatic vs manual deployment
- Set retention policies per phase

**Progression Rules**:
- "All must complete" before advancing
- "Minimum of X must complete"
- "Optional" phases can be skipped

**Common Lifecycles**:
- Default: Dev → Test → Staging → Production
- Hotfix: Staging → Production (skip lower environments)
- Feature Branch: Dev only

### 11. Channels

Allow different release strategies within a single project:
- Associate different lifecycles (hotfix vs standard)
- Apply package version rules (beta tags, version ranges)
- Scope variables and steps to specific channels
- Filter which tenants receive releases

**Use Cases**:
- Feature branch releases to test environments only
- Beta program releases to specific customers
- Hotfixes deployed directly to production

### 12. Tenants

Deploy to multiple customers, regions, or instances without duplicating projects:

**Tenant Features**:
- Connect tenants to projects and environments
- Tenant-specific variables (database connection, branding)
- Tenant tags for grouping (regions, tiers, release rings)
- Per-tenant infrastructure (dedicated or shared targets)
- Tenant-aware lifecycles

**Use Cases**:
- SaaS applications with per-customer instances
- Multi-region deployments
- Release rings (alpha → beta → general availability)

### 13. Runbooks

Automate operations tasks that aren't deployments:

**Characteristics**:
- No release required
- Lifecycles don't apply
- Can run on schedules or triggers
- Separate permissions from deployments

**Use Cases**:
- Infrastructure provisioning (Terraform, CloudFormation)
- Database backups and restores
- Certificate renewals
- Emergency failovers
- Server restarts

### 14. Packages and Feeds

**Packages**: Archives containing application files
- Formats: NuGet (.nupkg), ZIP, TAR, Docker images, Helm charts
- Versioned using SemVer
- Stored in package repositories

**Feeds**: Package repositories
- **Built-in Feed**: Octopus's internal repository
- **External Feeds**: NuGet, Docker registries, Maven, GitHub, Artifactory

## Deployment Patterns

### Rolling Deployments
Deploy to servers one-by-one instead of all at once:
- Configure window size (how many targets concurrently)
- Use child steps for complex sequences
- Maintains availability during deployment

### Blue-Green Deployments
Maintain two production environments (blue and green):
- Deploy to inactive environment
- Test the deployment
- Switch traffic to new environment
- Keep old environment for quick rollback

### Canary Deployments
Release to a small subset of targets/users first:
- Monitor for issues
- Gradually expand if successful
- Roll back quickly if problems arise

### Multi-Region Deployments
Deploy to multiple geographic regions:
- Use tenants or target tags for regions
- Region-specific variables and infrastructure
- Parallel or sequential regional rollouts

## Configuration as Code

Store project configuration in Git repositories:
- Deployment process as OCL (Octopus Configuration Language)
- Non-sensitive variables
- Runbook processes

**Benefits**:
- Version history and audit trail
- Branch-based development
- Code review for process changes
- GitOps workflows

## Security Model

### Authentication
- Username/Password
- Active Directory
- Azure AD, Okta, Google
- SAML 2.0

### Authorization
- **Teams**: Groups of users
- **User Roles**: Permissions sets (Project Deployer, Environment Manager)
- **Scoping**: Restrict roles to specific projects, environments, or tenants

### Communication Security
- Octopus ↔ Tentacle uses TLS with mutual certificate authentication
- Certificates generated on installation (2048-bit RSA)
- No passwords exchanged

## AI-Powered Capabilities

Octopus Deploy integrates AI to enhance continuous delivery workflows:

### Octopus AI Assistant
Intelligent companion integrated into the Octopus interface:
- **Tier-0 Support**: Instant answers from documentation via natural language
- **Project Creation**: Generate complete projects from text descriptions
- **Failure Analysis**: Analyze failed deployments and suggest remediation
- **Best Practices Adviser**: Identify unused variables, optimization opportunities

### Octopus MCP Server
Model Context Protocol server for AI assistant integration:
- Connect AI clients (Claude, etc.) to Octopus Deploy
- Combine with other MCP servers for complex workflows
- Change management, troubleshooting, and compliance use cases

### Recovery Agent (Cloud)
AI-powered rapid recovery from deployment failures:
- Automatic failure analysis
- Root cause identification
- Remediation suggestions

**Security**: Customer data never used to train models. Sensitive variables never exposed to AI. All interactions auditable.

## REST API

Octopus is API-first - everything in the UI is available via API:
- Hypermedia-driven (HATEOAS)
- Space-aware endpoints: `/api/{spaceId}/projects`
- API keys for authentication

**Octopus CLI**: Command-line interface for common operations
- Create releases
- Deploy releases
- Push packages
- Manage resources

### Octopus Variable Substitution Syntax
Octopus uses #{VariableName} syntax for variable substitution in steps and templates:

Basic: #{MyVariable}
Nested: #{Outer#{Inner}}
Escaped(literal) : ##{NotReplaced} renders as #{NotReplaced}

Conditionals:
#{if Octopus.Environment.Name == "Production"}production config#{/if}
#{if EnableFeature}enabled#{else}disabled#{/if}
#{unless IsDisabled}active#{/unless}

Falsy values: undefined, empty string, "False", "No", "0"

Iteration(comma-separated or indexed variables) :
#{each server in Servers}
    - #{server}
#{/each}
Loop variables: Octopus.Template.Each.Index, .First, .Last

Filters: #{Octopus.Environment.Name | ToLower}
Available: ToLower, ToUpper, ToBase64, FromBase64, HtmlEscape, XmlEscape, JsonEscape, Markdown, Trim, Truncate, Replace

Index lookup: #{MyPassword[#{UserName}]}
Calculations: #{calc Value1 + Value2}

### Output Variables (Passing Data Between Steps)
Scripts can set output variables that subsequent steps can read.

Reading output from a previous step uses the pattern:
Octopus.Action[StepName].Output.VariableName

In variable binding/substitution:
#{Octopus.Action[StepA].Output.TestResult}

## Octopus UI navigation

When directing user to navigate Octopus Deploy UI, refer to the following locations accurately:

<Menu>
<Deploy level=1>
Tenants
Tenant Tag Sets
Variable Sets
<Infrastructure level=2>
Overview
Deployment Targets
Environments
Machine Policies
Machine Proxies
Workers
Worker Pools
</Infrastructure>
<Manage level=2>
Accounts
Build Information
Certificates
External Feeds
Git Credentials
Lifecycles
Packages
Script Modules
Step Templates
</Manage>
</Deploy>

<SpecificProject level=2 name="{ReplaceWithProjectName}">
Dashboard
Process
Channels
Releases
Feature Toggles
Triggers
Freezes
Settings
Operations
Runbooks
Runbook Triggers
Ephemeral Environments
Project Variables
Tenant Variables
Variable Sets
All Variables
Variable Preview
Tenants
Tasks
Insights
Project Settings
Version Control
</SpecificProject>
<Insights level=1>
</Insights>
<PlatformHub level=1>
</Platform Hub>
<Configuration level=1 description="Octopus Instance Configuration">
</Configuration>
</Menu>

## Navigation Path Instructions

When directing users to UI locations, you MUST follow these formatting rules:

1. **Use the actual project name from context**: The `<SpecificProject name="{ReplaceWithProjectName}">` placeholder means you should use the ACTUAL project name provided in the OctopusContext. Never say "ReplaceWithProjectName" or "In" - use the real project name.

2. **Format navigation paths with arrows**: Use the format "ItemName -> SubItem -> Action"
   - Top-level items (level=1): Deploy, Insights, PlatformHub, Configuration
   - Sub-items (level=2): Infrastructure, Manage, and the project-specific menu
   - Project items: Process, Channels, Releases, Settings, Runbooks, etc.

3. **Examples of CORRECT navigation instructions**:
   - "MyProject -> Process -> step 'Run a Script'" (when project name is "MyProject")
   - "Acme Website -> Process -> step 'Deploy to Azure'" (when project name is "Acme Website")
   - "Deploy -> Infrastructure -> Deployment Targets"
   - "Deploy -> Manage -> Accounts"
   - "HelloWorld -> Project Variables"

4. **Examples of INCORRECT navigation instructions**:
   - "In -> Process -> step Run a Script" ❌ (Don't use "In" as a prefix)
   - "{ReplaceWithProjectName} -> Process" ❌ (Don't use the literal placeholder)
   - "Project -> Process" ❌ (Use the actual project name, not "Project")
   - "the Process tab" ❌ (Use full navigation path with arrows)

5. **When to bold entity names**: Bold the project name, step names, and other specific entity names in your instructions for clarity. Example: "Edit step **Run a Script** in **MyProject** -> **Process**"

## Related Knowledge Files

For detailed information on specific topics, refer to these companion files:

- [octopus-infrastructure.md](./octopus-infrastructure.md) - Environments, targets, workers, tentacles
- [octopus-projects.md](./octopus-projects.md) - Projects, processes, steps, variables
- [octopus-releases.md](./octopus-releases.md) - Releases, lifecycles, channels, deployments
- [octopus-runbooks.md](./octopus-runbooks.md) - Runbooks and operations automation
- [octopus-tenants.md](./octopus-tenants.md) - Multi-tenancy and tenant management
- [octopus-patterns.md](./octopus-patterns.md) - Deployment patterns and best practices
- [octopus-api.md](./octopus-api.md) - REST API, CLI, and automation
- [octopus-ai.md](./octopus-ai.md) - AI Assistant, MCP Server, Recovery Agent
- [Public Documentation](https://github.com/OctopusDeploy/docs) - Comprehensive user documentation and best practices
