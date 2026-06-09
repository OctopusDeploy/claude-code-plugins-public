# Octopus Deploy Tenants

This document details multi-tenancy in Octopus Deploy for deploying to multiple customers, regions, or instances.

## What are Tenants?

Tenants represent distinct deployment destinations that share the same deployment process but may have different configuration, infrastructure, or timing.

## Common Tenant Representations

### Customers
SaaS applications with per-customer instances:
- Each customer has dedicated infrastructure
- Customer-specific configuration (branding, features)
- Independent deployment schedules

### Regions
Multi-region deployments:
- US, EU, APAC deployments
- Region-specific compliance requirements
- Regional infrastructure

### Environments/Instances
Multiple instances of the same application:
- Development teams with own environments
- Feature branch deployments
- Testing instances

### Release Rings
Progressive rollout strategies:
- Alpha testers
- Beta program participants
- General availability

## Tenant Configuration

### Creating Tenants

**Required Information:**
- Tenant name
- Optional: Description, logo

**Connecting Projects:**
Link tenants to projects they should receive deployments for.

**Connecting Environments:**
Define which environments the tenant is deployed to.

### Tenant Variables

Configure tenant-specific values:

**Project Template Variables:**
Defined in project, values provided per tenant:
- Database connection string
- API endpoint
- Feature flags
- Branding settings

**Common Variables:**
Shared across projects (via Library Variable Sets):
- Tenant contact email
- Billing information
- SLA tier

### Variable Templates

Project-level templates that tenants fill in:

**Definition (Project):**
```
Variable: Database.ConnectionString
Type: Text
Required: Yes
Default: (none)
```

**Value (Tenant):**
```
Tenant: Acme Corp
Project: Web App
Database.ConnectionString = Server=acme-db;Database=AcmeApp
```

## Tenant Tags

Categorize and group tenants for bulk operations.

### Tag Sets

Collections of related tags:

**Example Tag Sets:**
```
Release Ring
├── Alpha
├── Beta
└── General Availability

Region
├── US-East
├── US-West
├── EU
└── APAC

Tier
├── Enterprise
├── Professional
└── Starter
```

### Tag Usage

**Channel Filtering:**
Limit releases to tenants with specific tags:
- Beta channel → Release Ring/Beta tenants
- Hotfix channel → Tier/Enterprise tenants

**Variable Scoping:**
Scope variables to tenant tags:
- Different feature flags by tier
- Regional configuration

**Deployment Filtering:**
Deploy to tenants matching tags:
- Deploy to all EU tenants
- Deploy to Alpha ring only

### Tenant Tag Inheritance

Tags can be hierarchical:
- Tenant inherits parent tag behaviors
- Useful for nested categorization

## Tenant Infrastructure

### Dedicated Infrastructure
Each tenant has dedicated deployment targets:
- Tenant-specific machines
- Isolated resources

**Configuration:**
1. Add deployment targets
2. Tag targets with tenant-specific roles
3. Connect targets to tenant

### Shared Infrastructure
Multiple tenants deploy to same targets:
- Cost efficiency
- Shared resources with logical separation

**Configuration:**
1. Create shared targets
2. Associate with multiple tenants
3. Manage concurrency

### Hybrid Model
Mix of dedicated and shared:
- Enterprise tier: Dedicated
- Starter tier: Shared

## Tenanted Deployments

### Deployment Modes

**Untenanted Deployments:**
Traditional deployment without tenant context.

**Tenanted Deployments:**
Deployment with specific tenant(s) selected.

**Project Settings:**
- Require tenanted deployments
- Allow both tenanted and untenanted
- Disable tenanted deployments

### Deployment Scenarios

**Single Tenant:**
Deploy release 1.2.3 to Production for Acme Corp

**Multiple Tenants:**
Deploy release 1.2.3 to Production for all EU tenants

**All Tenants:**
Deploy release 1.2.3 to Production for all connected tenants

### Deployment Concurrency

**Default:** Tenanted deployments run in parallel

**Serial Deployment:**
Control with concurrency tag:
```
Octopus.Task.ConcurrencyTag = #{Octopus.Project.Id}/#{Octopus.Environment.Id}
```

### Tenant-Aware Lifecycles

Control deployment progression per tenant.

**Standard:**
All tenants follow same lifecycle phases.

**Tenant-Specific:**
Use tenant tags to vary deployment timing:
- Alpha tenants: Auto-deploy to Test
- GA tenants: Manual approval required

## Tenant Variables Deep Dive

### Variable Precedence

When both project and tenant define values:
```
Tenant Variable > Project Variable > Library Variable
```

### Tenant Variables Not Snapshotted

**Important:** Tenant variables are NOT captured in releases.

**Implications:**
- Tenant variable changes take immediate effect
- New tenants can be deployed without new releases
- Existing releases work with tenant changes

### Sensitive Tenant Variables

Mark sensitive for encryption:
- API keys
- Credentials
- Secrets

### Variable Prompting

Combine prompted variables with tenants:
- Prompt once, deploy to multiple tenants
- Tenant-specific prompts

## Tenant Permissions

### Permission Scoping

Restrict team access to specific tenants:
- Team A: Manages tenants with Tag X
- Team B: Manages specific tenant list

### Tenant Security Patterns

**Customer-Isolated Access:**
Customer support teams only see their customer's tenant.

**Regional Teams:**
Regional ops teams manage their region's tenants.

**Tier-Based Access:**
Enterprise support team handles enterprise tenants.

## Multi-Tenant Patterns

### SaaS Multi-Tenancy

**Scenario:** Each customer has isolated deployment

**Setup:**
1. Create tenant per customer
2. Define customer-specific variables
3. Connect to projects
4. Deploy independently

### Regional Deployments

**Scenario:** Deploy to multiple geographic regions

**Setup:**
1. Create tenant per region (or region as tag)
2. Region-specific infrastructure
3. Staggered regional rollouts
4. Regional compliance variables

### Release Rings

**Scenario:** Progressive rollout strategy

**Setup:**
1. Create tag set: Release Ring
2. Tags: Alpha, Beta, GA
3. Assign tenants to rings
4. Configure channels with tenant filters
5. Deploy progressively through rings

### Development Teams

**Scenario:** Teams have own environments

**Setup:**
1. Tenant per team
2. Team-specific infrastructure
3. Self-service deployment capability

## Best Practices

### Tenant Design
- Plan tenant strategy before implementation
- Consider growth patterns
- Document tenant purpose

### Tag Strategy
- Use meaningful tag sets
- Keep tag hierarchies simple
- Plan for filtering needs

### Variables
- Define templates at project level
- Validate tenant variable completeness
- Use sensitive marking appropriately

### Permissions
- Follow principle of least privilege
- Group tenants logically for permission scoping
- Regular access reviews

### Scaling
- Anticipate tenant growth
- Automate tenant provisioning
- Monitor tenant-specific metrics

## Tenant Lifecycle

### Tenant Onboarding
1. Create tenant
2. Configure variables
3. Connect to projects
4. Assign infrastructure
5. Test deployment
6. Go live

### Tenant Offboarding
1. Disable deployments
2. Archive/remove infrastructure
3. Retain audit history
4. Remove tenant (optional)

### Tenant Maintenance
- Update variables as needed
- Rotate credentials
- Review permissions
- Monitor deployment health
