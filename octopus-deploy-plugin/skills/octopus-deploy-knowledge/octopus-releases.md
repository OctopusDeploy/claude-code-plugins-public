# Octopus Deploy Releases, Lifecycles, and Channels

This document details how releases work in Octopus Deploy, including lifecycles, channels, and deployment execution.

## Releases

A release is an immutable snapshot of everything required for a deployment.

### What's Captured in a Release

- **Deployment Process**: Steps and their configuration
- **Variable Snapshot**: All project and library variable values (except tenant variables)
- **Package Versions**: Specific versions of all packages referenced
- **Channel**: Which channel the release belongs to

### Release Immutability

**Key Principle**: Once created, a release doesn't change.

- Modifications to the deployment process don't affect existing releases
- Variable changes don't affect existing releases (except tenant variables)
- Same release deployed to Dev and Production uses identical process

**Why This Matters:**
- Predictable deployments
- What worked in Test works in Production
- Audit trail accuracy
- Reliable rollbacks

### Release Versioning

Releases require version numbers, typically following SemVer:

**Semantic Versioning Format:**
```
Major.Minor.Patch[-PreRelease][+Build]
1.2.3-beta.1+build.456
```

**Version Components:**
- **Major**: Breaking changes
- **Minor**: New features, backwards compatible
- **Patch**: Bug fixes
- **PreRelease**: Alpha, beta, RC identifiers
- **Build**: Build metadata

**Auto-Versioning:**
- Based on latest package version
- Templated versioning
- Build server integration

### Creating Releases

**Manual Creation:**
1. Navigate to Project → Releases → Create Release
2. Select channel
3. Choose package versions
4. Enter version number
5. Add release notes
6. Create

**Automatic Creation:**
- Built-in package repository triggers
- External feed triggers
- Git triggers (for version-controlled projects)
- Build server plugins

### Release Notes

Document what's in each release:
- Manual entry
- Generated from package build information
- Linked to work items (Jira, Azure DevOps, GitHub)

## Deployments

A deployment is the execution of a release to an environment.

### Deployment Properties

- **Release**: Which release to deploy
- **Environment**: Target environment
- **Tenant**: (Optional) Specific tenant
- **Timing**: Immediate or scheduled
- **Target Selection**: All targets or specific machines

### Deployment Execution

**Process Flow:**
1. Select release and environment
2. Resolve deployment targets via tags
3. Acquire packages
4. Execute steps in order
5. Capture logs and artifacts
6. Report success/failure

### Deployment Concurrency

**Default Behavior:**
Deployments to the same project/environment/tenant run serially.

**Concurrency Tag:**
Control with `Octopus.Task.ConcurrencyTag` variable:
```
#{Octopus.Project.Id}/#{Octopus.Environment.Id}/#{Octopus.Deployment.Tenant.Id}
```

### Guided Failures

When enabled, deployment failures pause for intervention:

**Options:**
- **Fail**: Terminate the deployment
- **Retry**: Re-run the failed step
- **Ignore**: Continue as if successful
- **Exclude Machine**: Skip this target, continue with others

### Deployment Targets Selection

**Default:** Deploy to all healthy targets matching step tags

**Exclusions:**
- Exclude specific machines
- Skip unhealthy/unavailable targets

**Machine-Specific Deployments:**
Deploy to specific machines only (useful for troubleshooting)

## Lifecycles

Lifecycles control release progression through environments.

### Lifecycle Structure

```
Lifecycle: Standard
├── Phase 1: Development
│   └── Environment: Dev (Auto-deploy)
├── Phase 2: Testing
│   └── Environment: Test (Manual)
├── Phase 3: Pre-Production
│   └── Environment: Staging (Manual)
└── Phase 4: Production
    └── Environment: Prod (Manual)
```

### Phases

A phase is a stage in the lifecycle containing one or more environments.

**Phase Properties:**
- **Name**: Phase identifier
- **Environments**: Which environments are in this phase
- **Required to Progress**: How many must complete
- **Automatic Deployment**: Deploy automatically when phase becomes active

### Progression Rules

**All Must Complete:**
Every environment in the phase must have a successful deployment.

**Minimum of X:**
At least X environments must succeed before advancing.

**Optional:**
Phase can be skipped entirely.

### Automatic Deployments

**Per-Environment Setting:**
Each environment in a phase can auto-deploy or require manual triggering.

**Tenant Behavior:**
Automatic deployments in tenanted scenarios:
1. Filter tenants by channel's tenant filter
2. Apply lifecycle progression rules
3. Enqueue deployments for matching tenants

### Default Lifecycle

**Behavior:**
- Creates implicit phases from environments
- One phase per environment
- Order matches environment order

**Recommendation:**
Define explicit phases rather than relying on defaults.

### Common Lifecycle Patterns

**Standard Pipeline:**
```
Dev → Test → Staging → Production
```

**Hotfix Pipeline:**
```
Staging → Production (skip Dev/Test)
```

**Feature Branch:**
```
Dev only (or Feature environment)
```

**Multi-Region:**
```
Dev → Test → Staging → Prod-US → Prod-EU → Prod-APAC
```

### Retention Policies

Control how long releases and deployed files are kept.

**Release Retention:**
- Keep X releases per lifecycle phase
- Keep releases for X days

**Tentacle Retention:**
- Keep X deployed packages on targets
- Clean up old extracted packages

**Configuration:**
- Set at lifecycle level
- Override per phase

## Channels

Channels enable different release strategies within a single project.

### Channel Purpose

- Associate different lifecycles with the same project
- Apply package version rules
- Scope variables and steps
- Filter tenant deployments

### Channel Types

**Lifecycle Channels:**
Standard channels that progress through lifecycle phases.

**Ephemeral Environment Channels:**
Create temporary environments per release (e.g., per pull request).

### Package Version Rules

Control which package versions can be used in a channel.

**Version Range:**
```
[1.0,2.0)  - 1.x versions only
[2.0,)    - 2.0 and above
```

**Pre-Release Tags:**
```
^beta      - Versions starting with beta
^$         - No pre-release tag (stable only)
^(?!beta)  - Not starting with beta
```

### Git Protection Rules

For version-controlled projects:

**Branch Rules:**
```
main, release/*   - Only these branches
```

**Tag Rules:**
```
v[0-9].*  - Version tags only
```

### Channel Use Cases

**Feature Branches:**
- Channel: `Feature`
- Lifecycle: Dev only
- Branch rules: `feature/*`

**Hotfix Releases:**
- Channel: `Hotfix`
- Lifecycle: Staging → Production
- Version rules: `hotfix-*` tags

**Early Access:**
- Channel: `Beta`
- Lifecycle: Dev → Test → Beta-Environment
- Tenant filter: Early Adopter tenants

### Default Channel

Every project has at least one channel (Default).

**Behavior:**
- Used when no other channel matches
- Inherits project's default lifecycle
- No version rules by default

### Channel Selection

**Manual:**
Select channel when creating release.

**Automatic:**
Octopus selects based on package versions matching channel rules.

## Deployment Changes

Track what changed between deployments.

### Change Tracking

**Includes:**
- All releases between current and previous deployment
- Commit history (from build information)
- Work items (from issue tracker integrations)

### Build Information

Package metadata pushed from build servers:
- Commits included in build
- Work item links
- Build URL
- Branch name

### Issue Tracker Integration

**Supported Systems:**
- Jira
- Azure DevOps
- GitHub Issues

**Functionality:**
- Parse commit messages for issue IDs
- Link deployments to work items
- Generate release notes from issues

## Release Progression Blocking

### Prevent Release Progression

Block a release from advancing to higher environments.

**Use Cases:**
- Known bug discovered
- Failed compliance check
- Pending security review

### Guided Failure Recovery

When using guided failures:
- Pause allows investigation
- Retry after fixes
- Exclude problematic targets
- Fail to prevent further progression

## Best Practices

### Release Management
- Use meaningful version numbers
- Document releases with notes
- Link to work items
- Test in lower environments before production

### Lifecycle Design
- Keep phase count reasonable
- Use automatic deployment for Dev
- Require manual approval for Production
- Set appropriate retention policies

### Channel Strategy
- Use channels for distinct release strategies
- Don't overuse channels
- Keep version rules simple
- Document channel purposes

### Deployment Safety
- Enable guided failures for production
- Exclude unhealthy targets
- Monitor deployment progress
- Have rollback plans ready
