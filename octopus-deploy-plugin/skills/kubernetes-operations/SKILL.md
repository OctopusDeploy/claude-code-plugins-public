---
name: kubernetes-operations
description: Query and troubleshoot Kubernetes deployments, check live pod status, and monitor Kubernetes resources
allowed-tools:
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__get_kubernetes_live_status
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__list_deployments
  - mcp__plugin_octopus-deploy-devops_octopusdeploy__get_deployment_process
---

# Kubernetes Operations

Use this skill to query and troubleshoot Kubernetes deployments managed by Octopus Deploy, including live pod status and resource health.

## Important: Version Requirement

This skill requires Octopus Server **2025.3 or later**. The `get_kubernetes_live_status` tool is not available on earlier versions.

## When to Use This Skill

- Checking live Kubernetes pod status
- Reviewing Kubernetes resource health
- Troubleshooting container issues
- Monitoring deployment rollouts
- Validating Kubernetes deployments
- Investigating pod failures or crashes

## Available Tools

This skill provides access to Kubernetes operations tools:
- Get live status of Kubernetes resources (pods, deployments, services)
- List Kubernetes deployments from Octopus
- Review Kubernetes deployment processes
- Monitor pod health and availability

## Common Workflows

See [k8s-troubleshooting.md](k8s-troubleshooting.md) for Kubernetes troubleshooting patterns.

See [live-status-queries.md](live-status-queries.md) for live status query examples.

## Required Setup

### Environment Variables
Ensure these environment variables are configured:
- `OCTOPUS_SERVER_URL`: Your Octopus server URL (must be 2025.3+)
- `OCTOPUS_API_KEY`: Your Octopus API key with read access

### Octopus Server Version
Verify your Octopus Server version supports Kubernetes live status:
- Minimum required version: **2025.3**
- Check your server version before using this skill

## Use Cases

### Pod Health Monitoring
- "Show me the status of all pods in namespace X"
- "Are all pods healthy for deployment Y?"
- "Which pods are crashing or in error state?"

### Resource Troubleshooting
- "Why are pods failing to start?"
- "What's the restart count for these pods?"
- "Show me pod events and errors"

### Deployment Validation
- "Is the deployment rollout complete?"
- "How many replicas are ready?"
- "Are there any pending pods?"

### Container Investigation
- "What's the status of container X in pod Y?"
- "Why is this container in a waiting state?"
- "Show me recent container restarts"
