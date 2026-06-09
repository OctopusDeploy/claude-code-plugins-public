# Octopus Deploy Infrastructure

This document details the infrastructure components in Octopus Deploy: environments, deployment targets, workers, and communication mechanisms.

## Environments

Environments represent the stages in your deployment pipeline where software is deployed and run.

### Purpose

- Group deployment targets by deployment stage
- Control release progression through lifecycles
- Scope variables to specific stages
- Apply permissions per environment

### Common Environment Structure

```
Development → Test/QA → Staging → Production
```

**Typical Environments:**

| Environment | Purpose |
|-------------|---------|
| Development | Developer experimentation, frequently in flux |
| Test/QA | Quality assurance testing |
| Staging/Pre-Prod | Final validation, mirrors production |
| Production | End-user facing systems |

### Environment Ordering

Environments are ordered from least to most production-like:
- Affects dashboard display order
- Determines default lifecycle progression
- Influences release deployment order

### Environment Configuration

**Dynamic Infrastructure**: Enable/disable ability to create deployment targets programmatically during deployments.

**Guided Failure Mode**: When enabled, deployment failures pause and allow user intervention:
- Retry the failed step
- Ignore the failure
- Fail the deployment
- Exclude the machine and continue

### Environment Tags

Classify environments with custom tags for:
- Cloud provider identification
- Regional grouping
- Tier classification
- Dashboard filtering

## Deployment Targets

A deployment target is any destination where Octopus deploys software.

### Target Types

#### 1. Tentacle (Windows and Linux)

Lightweight agent installed on Windows or Linux servers.

**Listening Tentacle (Recommended)**
- Tentacle listens on a TCP port (default: 10933)
- Octopus Server initiates connections
- Best for: Most scenarios, simpler firewall rules
- Requires: Inbound port open on target

**Polling Tentacle**
- Tentacle polls Octopus Server for work
- Tentacle initiates all connections
- Best for: Targets behind NAT/firewalls, DMZ scenarios
- Requires: Outbound HTTPS to Octopus Server

**Communication Security:**
- All communication over TLS
- Mutual certificate authentication
- 2048-bit RSA certificates generated on installation
- No passwords exchanged

#### 2. SSH Targets

Deploy to Linux/Unix machines via SSH.

**Authentication Methods:**
- Username/Password
- SSH Key Pair

**Execution Agent (Calamari):**
- Automatically deployed on first use
- Self-contained .NET runtime
- No Mono dependency required

#### 3. Kubernetes Targets

**Kubernetes Agent (Recommended)**
- Installed via Helm chart in the cluster
- Self-registers with Octopus
- Runs deployments inside the cluster
- Supports high-availability clusters
- Better network isolation

**Kubernetes API Target**
- Direct API access from workers
- Requires cluster credentials
- Useful when agent installation isn't possible

**Supported Distributions:**
- Vanilla Kubernetes
- AKS, EKS, GKE
- OpenShift
- Rancher

#### 4. Cloud Targets

**Azure:**
- Azure Web Apps
- Azure Cloud Services
- Azure Service Fabric

**AWS:**
- ECS clusters
- Lambda functions (via steps)

#### 5. Offline Package Drop

Generate deployment packages for manual deployment in air-gapped environments.

### Target Tags (Roles)

Logical groupings that associate targets with deployment steps.

**Purpose:**
- Steps target tags, not individual machines
- Add/remove machines without modifying deployment process
- Support dynamic infrastructure

**Examples:**
- `web-server` - Web application hosts
- `app-server` - Application tier servers
- `db-server` - Database servers
- `frontend`, `backend`, `api`

**Best Practices:**
- Use descriptive, function-based names
- Multiple tags per target are allowed
- Avoid overly specific tags

### Health Checks

Periodic checks to verify target availability.

**Health Statuses:**
- **Healthy**: Target responsive and operational
- **Unhealthy**: Target has issues but may be usable
- **Unavailable**: Target cannot be contacted
- **Unknown**: Health check hasn't run or errored

**Machine Policies:**
- Configure health check intervals
- Define Calamari update behavior
- Set connectivity timeouts
- Assign PowerShell/Bash health check scripts

### Dynamic Infrastructure

Create and remove deployment targets programmatically during deployments.

**PowerShell Functions:**
```powershell
# Create a new target
New-OctopusTarget -Name "WebServer01" -TargetRole "web-server" ...

# Remove a target
Remove-OctopusTarget -Name "WebServer01"
```

**Use Cases:**
- Auto-scaling environments
- Ephemeral infrastructure (containers, VMs)
- Infrastructure-as-Code provisioning

## Workers

Machines that execute deployment tasks without being deployment targets.

### Purpose

- Run scripts and steps that don't target specific infrastructure
- Execute AWS, Azure, Terraform, Kubernetes steps
- Offload work from Octopus Server
- Provide tools and dependencies

### Worker Types

#### Built-in Worker (Self-Hosted Only)
- Runs on the Octopus Server machine
- Default for steps not targeting specific machines
- Disabled when external workers are added to default pool

**Security Consideration:** Executes under Octopus Server's service account. Consider external workers for sensitive operations.

#### Dynamic Workers (Octopus Cloud)
- On-demand VMs provisioned by Octopus
- Available OS images: Ubuntu, Windows
- Pre-configured with common tools
- No infrastructure management required

#### External Workers
- Self-managed Tentacles or SSH machines
- Registered to Worker Pools
- Full control over tools and configuration

### Worker Pools

Group workers by purpose, location, or capability.

**Use Cases:**
- Network zone isolation (DMZ, internal)
- Tool-specific workers (Java, .NET, Node.js)
- Region-specific workers
- Performance isolation

**Default Worker Pool:**
- Steps default to this pool
- Adding workers here disables built-in worker

### Worker Selection

Octopus selects workers from pools based on:
- Pool availability
- Worker health status
- Package caching (reuses workers with cached packages)

**Package Affinity:**
By default, Octopus reuses workers that have downloaded required packages. Override with `Octopus.Deployment.WorkerLeaseCap` variable.

## Accounts

Credentials for authenticating with external services during deployments.

### Account Types

**Azure Accounts:**
- Service Principal (recommended)
- Management Certificate (legacy)

**AWS Accounts:**
- Access Key/Secret Key
- IAM Role assumption

**Google Cloud Accounts:**
- Service Account JSON key

**SSH Key Pair:**
- Private key for SSH authentication

**Token Accounts:**
- API tokens for services

**Username/Password:**
- Basic authentication credentials

### Account Security

- Stored encrypted in database
- Master key encryption
- Scoped to environments (optional)
- Audit trail for usage

## Tentacle Communication Details

### Trust Relationship

1. Tentacle is configured with Octopus Server's certificate thumbprint
2. Octopus is given Tentacle's certificate thumbprint
3. Both parties verify identity via certificates

### Listening Tentacle Flow

1. Octopus establishes TLS connection to Tentacle
2. Tentacle presents server certificate
3. Octopus presents client certificate
4. Connection held open for commands

### Polling Tentacle Flow

1. Tentacle establishes TLS connection to Octopus
2. Octopus presents server certificate
3. Tentacle presents client certificate
4. Tentacle polls for pending commands

### TLS Requirements

- Minimum: TLS 1.2
- Supported: TLS 1.3 (on modern OS)
- Protocol negotiated based on OS capabilities

### Proxy Support

Both Octopus Server and Tentacle support:
- HTTP proxies for web requests
- Proxy authentication
- Proxy bypass rules

## Infrastructure Best Practices

### Environment Design
- Keep environment count manageable (under 10)
- Order from least to most production-like
- Use consistent naming conventions

### Target Tags
- Use role-based, not machine-based names
- Consider deployment topology
- Plan for scaling

### Workers
- Use external workers for security-sensitive operations
- Match worker capabilities to step requirements
- Consider network proximity to targets

### Health Checks
- Configure appropriate check intervals
- Monitor health status dashboard
- Investigate unhealthy targets promptly

### Security
- Use unique certificates per target
- Restrict Tentacle to specific Octopus thumbprints
- Limit service account permissions
- Regular certificate rotation for high-security environments
