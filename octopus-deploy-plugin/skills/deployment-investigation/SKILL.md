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
- List deployments with filtering by project, environment, and status
- Get detailed task information including logs and error messages
- Query deployment history across spaces and environments
- Review raw task details for deep troubleshooting

## Common Workflows

See [troubleshooting-guide.md](troubleshooting-guide.md) for step-by-step troubleshooting patterns.

See [example-queries.md](example-queries.md) for example queries and investigation approaches.

## Required Setup

Ensure these environment variables are configured:
- `OCTOPUS_SERVER_URL`: Your Octopus server URL
- `OCTOPUS_API_KEY`: Your Octopus API key with read access
