---
name: release-management
description: Manage and analyze releases across projects, review deployment processes, and track Git branch deployments
allowed-tools:
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__list_releases
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__list_releases_for_project
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__get_release_by_id
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__get_deployment_process
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__get_branches
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__list_projects
---

# Release Management

Use this skill to manage and analyze releases, review deployment processes, and track releases across projects and environments.

## When to Use This Skill

- Finding the latest release for a project
- Comparing release versions across environments
- Reviewing deployment process steps
- Tracking Git branch deployments
- Analyzing release history and patterns
- Understanding what's included in a release

## Available Tools

This skill provides access to release management tools:
- List all releases or releases for specific projects
- Get detailed release information including packages and variables
- Review deployment processes and step configurations
- Query Git branches for version-controlled projects (Octopus 2021.2+)
- Track release progression across environments

## Common Workflows

See [release-workflows.md](release-workflows.md) for step-by-step release management patterns.

See [git-branch-patterns.md](git-branch-patterns.md) for Git branch deployment strategies.

## Required Setup

Ensure these environment variables are configured:
- `OCTOPUS_SERVER_URL`: Your Octopus server URL
- `OCTOPUS_API_KEY`: Your Octopus API key with read access

## Version Requirements

- `get_branches` requires Octopus Server 2021.2 or later
