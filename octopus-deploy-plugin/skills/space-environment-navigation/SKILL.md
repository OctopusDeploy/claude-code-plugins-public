---
name: space-environment-navigation
description: Navigate and understand Octopus structure across spaces, environments, and projects
allowed-tools:
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__list_spaces
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__list_environments
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__list_projects
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__get_current_user
---

# Space and Environment Navigation

Use this skill to navigate and understand the structure of your Octopus Deploy instance across spaces, environments, and projects.

## When to Use This Skill

- Exploring a new or unfamiliar Octopus instance
- Understanding the space and environment structure
- Mapping project-environment relationships
- Verifying user permissions and access
- Getting oriented in multi-space instances
- Discovering available projects and environments

## Available Tools

This skill provides access to navigation tools:
- List all spaces in the Octopus instance
- List environments within a space
- List projects and their configurations
- Get current user information and permissions

## Common Workflows

See [navigation-patterns.md](navigation-patterns.md) for step-by-step navigation and exploration patterns.

## Required Setup

Ensure these environment variables are configured:
- `OCTOPUS_SERVER_URL`: Your Octopus server URL
- `OCTOPUS_API_KEY`: Your Octopus API key with read access

## Use Cases

### Instance Discovery
- "What spaces exist in this Octopus instance?"
- "Show me all environments in the Production space"
- "List all projects in the Development space"

### Access Verification
- "What permissions do I have?"
- "Which spaces can I access?"
- "Am I an administrator?"

### Structure Understanding
- "How is this Octopus instance organized?"
- "What's the relationship between projects and environments?"
- "What environments are available for deployments?"
