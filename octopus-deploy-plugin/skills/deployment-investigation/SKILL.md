---
name: deployment-investigation
description: Investigate deployment failures, analyze task logs, and troubleshoot Octopus Deploy issues
allowed-tools:
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__list_deployments
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__get_task_by_id
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__get_task_details
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__get_task_raw
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__list_environments
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__list_projects
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__get_task_from_url
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__get_deployment_from_url
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__list_spaces
---

# Deployment Investigation

Use this skill to investigate deployment failures, analyze task logs, and troubleshoot issues in Octopus Deploy.

## When to Use This Skill

- Finding recent failed deployments
- Analyzing error messages and stack traces
- Comparing successful vs failed deployments
- Reviewing deployment task logs
- Investigating deployment issues across environments

## Available Tools

This skill provides access to deployment and task investigation tools:
- **URL-aware tools**: Extract information directly from Octopus URLs
  - `get_task_from_url`: Get task details from task or deployment URLs
  - `get_deployment_from_url`: Get deployment details from deployment URLs
- List deployments with filtering by project, environment, and status
- Get detailed task information including logs and error messages
- Query deployment history across spaces and environments
- Review raw task details for deep troubleshooting
- List spaces to resolve space IDs to names

## Working with URLs

When investigating deployments from URLs, follow the resource relationships correctly:

**Deployment URLs** → Extract deployment ID → Query deployment → Get task ID → Query task
**Task URLs** → Extract task ID directly → Query task

See [../../guides/working-with-urls.md](../../guides/working-with-urls.md) for comprehensive URL handling patterns and common pitfalls.

### Quick URL Workflow

1. **Deployment URL with task logs needed**:
   - Extract deployment ID (e.g., `Deployments-123`)
   - Use `list_deployments` to get deployment details
   - Extract `TaskId` from response
   - Use `get_task_details` with the task ID

2. **Task URL with logs needed**:
   - Extract task ID (e.g., `ServerTasks-456`)
   - Use `get_task_details` directly

3. **Space ID resolution**:
   - URLs contain space IDs (e.g., `Spaces-1`)
   - Tools need space names (e.g., `"Team A Space"`)
   - Use `list_spaces` to resolve ID to name

## Common Workflows

See [troubleshooting-guide.md](troubleshooting-guide.md) for step-by-step troubleshooting patterns.

See [example-queries.md](example-queries.md) for example queries and investigation approaches.

## Required Setup

Ensure these environment variables are configured:
- `OCTOPUS_SERVER_URL`: Your Octopus server URL
- `OCTOPUS_API_KEY`: Your Octopus API key with read access
