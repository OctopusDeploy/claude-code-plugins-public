---
name: infrastructure-audit
description: Audit deployment targets, certificates, accounts, and infrastructure health in Octopus Deploy
allowed-tools:
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__list_deployment_targets
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__get_deployment_target
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__list_certificates
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__get_certificate
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__list_accounts
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__get_accounts
---

# Infrastructure Audit

Use this skill to audit deployment targets, certificates, accounts, and overall infrastructure health in Octopus Deploy.

## When to Use This Skill

- Checking deployment target health and availability
- Reviewing certificate expiration dates
- Auditing account configurations and credentials
- Identifying offline or unhealthy targets
- Validating infrastructure before deployments
- Security compliance auditing

## Available Tools

This skill provides access to infrastructure audit tools:
- List and inspect deployment targets (machines, workers, cloud regions)
- Review certificates and expiration dates
- Audit accounts and their configurations
- Check target health status and connectivity

## Common Workflows

See [health-checks.md](health-checks.md) for infrastructure health monitoring patterns.

See [certificate-management.md](certificate-management.md) for certificate lifecycle management.

## Required Setup

Ensure these environment variables are configured:
- `OCTOPUS_SERVER_URL`: Your Octopus server URL
- `OCTOPUS_API_KEY`: Your Octopus API key with read access

## Use Cases

### Target Health Monitoring
- "Are all deployment targets online?"
- "Which targets are unhealthy or offline?"
- "What's the health status of production targets?"

### Certificate Management
- "Which certificates are expiring soon?"
- "Show me all certificates expiring in the next 30 days"
- "What certificates are used in production?"

### Account Auditing
- "List all cloud accounts"
- "What AWS accounts are configured?"
- "Show me Azure subscription configurations"

### Infrastructure Validation
- "Is the infrastructure ready for deployment?"
- "Are there any unhealthy targets in the environment?"
- "What accounts have access to production?"
