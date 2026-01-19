---
name: tenant-management
description: Manage multi-tenant deployments, review tenant configurations, and track tenant-specific variables
allowed-tools:
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__list_tenants
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__get_tenant_by_id
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__get_tenant_variables
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__get_missing_tenant_variables
---

# Tenant Management

Use this skill to manage multi-tenant deployments, review tenant configurations, and track tenant-specific variables in Octopus Deploy.

## When to Use This Skill

- Managing SaaS applications with multiple customers
- Reviewing tenant configurations and settings
- Identifying missing tenant variables
- Auditing tenant-specific deployments
- Validating tenant setup before deployments
- Tracking tenant variable values

## Available Tools

This skill provides access to tenant management tools:
- List all tenants with filtering options
- Get detailed tenant information and configuration
- Review tenant-specific variables (common, project-specific, or all)
- Identify missing or unfilled tenant variables

## Common Workflows

See [tenant-patterns.md](tenant-patterns.md) for multi-tenancy deployment patterns.

See [variable-management.md](variable-management.md) for tenant variable configuration.

## Required Setup

Ensure these environment variables are configured:
- `OCTOPUS_SERVER_URL`: Your Octopus server URL
- `OCTOPUS_API_KEY`: Your Octopus API key with read access

## Use Cases

### Tenant Configuration
- "List all tenants in the space"
- "Show me configuration for Customer-ABC tenant"
- "What tenants are connected to the WebApp project?"

### Variable Management
- "What variables are missing for tenant Customer-XYZ?"
- "Show me all tenant-specific variables for the API project"
- "Which tenants have incomplete configuration?"

### Deployment Tracking
- "What's deployed to tenant Customer-ABC?"
- "Which tenants are on version 2.0?"
- "Show me deployment history for this tenant"

### Tenant Validation
- "Is this tenant ready for production deployment?"
- "Are all required variables configured?"
- "What environments is this tenant connected to?"
