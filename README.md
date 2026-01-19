# Octopus Deploy Claude Code Plugins

Public Claude Code plugins for Octopus Deploy integrations, providing comprehensive skills and MCP integration for operations tasks.

## Features

- **MCP Integration**: Direct access to 24 read-only Octopus Deploy API tools via the official `@octopusdeploy/mcp-server`
- **6 Specialized Skills**: Comprehensive coverage of common Octopus Deploy operational workflows

## Installation

### 1. Add Marketplace and Install Plugin

```bash
/plugin marketplace add https://github.com/OctopusDeploy/claude-code-plugins-public
/plugin install octopus-deploy-devops
```

### 2. Configure Environment Variables

The plugin requires these environment variables to be configured:

```bash
export OCTOPUS_SERVER_URL="https://your-instance.octopus.app"
export OCTOPUS_API_KEY="API-XXXXXXXXXXXXX"
```

## Skills Overview

### Deployment Investigation
Investigate deployment failures, analyze task logs, and troubleshoot Octopus Deploy issues.

**Use Cases:**
- Find recent failed deployments
- Analyze error messages and stack traces
- Compare successful vs failed deployments
- Review deployment task logs

**Skills:** `/octopus-deploy-devops:deployment-investigation`

### Release Management
Manage and analyze releases across projects, review deployment processes, and track Git branch deployments.

**Use Cases:**
- Find latest releases for projects
- Compare release versions across environments
- Review deployment process steps
- Track Git branch deployments (requires Octopus 2021.2+)

**Skills:** `/octopus-deploy-devops:release-management`

### Space and Environment Navigation
Navigate and understand Octopus structure across spaces, environments, and projects.

**Use Cases:**
- Explore multi-space instances
- Map project-environment relationships
- Verify user permissions and access
- Understand instance organization

**Skills:** `/octopus-deploy-devops:space-environment-navigation`

### Infrastructure Audit
Audit deployment targets, certificates, accounts, and infrastructure health.

**Use Cases:**
- Check deployment target health and availability
- Review certificate expiration dates
- Audit account configurations
- Identify offline or unhealthy targets

**Skills:** `/octopus-deploy-devops:infrastructure-audit`

### Tenant Management
Manage multi-tenant deployments, review tenant configurations, and track tenant-specific variables.

**Use Cases:**
- Manage SaaS applications with multiple customers
- Review tenant configurations and settings
- Identify missing tenant variables
- Validate tenant setup before deployments

**Skills:** `/octopus-deploy-devops:tenant-management`

### Kubernetes Operations
Query and troubleshoot Kubernetes deployments, check live pod status, and monitor resources.

**Important:** Requires Octopus Server 2025.3 or later

**Use Cases:**
- Check live Kubernetes pod status
- Review resource health
- Troubleshoot container issues
- Monitor deployment rollouts

**Skills:** `/octopus-deploy-devops:kubernetes-operations`

## MCP Tools

The plugin provides access to read-only tools via the Octopus Deploy MCP server. For a complete list of available tools and their documentation, see the [Octopus Deploy MCP Server repository](https://github.com/OctopusDeploy/mcp-server).

## Usage Examples

### Investigate a Failed Deployment

```
/octopus-deploy-devops:deployment-investigation

"Show me all failed deployments in the last 24 hours for the Production space"
```

### Find Latest Release

```
/octopus-deploy-devops:release-management

"What's the latest release for the API project?"
```

### Check Infrastructure Health

```
/octopus-deploy-devops:infrastructure-audit

"Are all deployment targets healthy in the Production environment?"
```

### Review Tenant Configuration

```
/octopus-deploy-devops:tenant-management

"What variables are missing for tenant Customer-ABC?"
```

### Monitor Kubernetes Pods

```
/octopus-deploy-devops:kubernetes-operations

"Show me the status of all pods in the production namespace"
```

## Requirements

- Octopus Deploy instance (Cloud or Server)
- Octopus API key with read access
- Node.js (for MCP server via npx)
- Claude Code

### Version-Specific Requirements

- **Git branch queries**: Octopus Server 2021.2 or later
- **Kubernetes live status**: Octopus Server 2025.3 or later

## Troubleshooting

### MCP Server Connection Issues

If the MCP server fails to connect:

1. Verify API key permissions:
   - API key must have read access to required resources
   - Check API key expiration

2. Test the MCP server manually:
   ```bash
   npx -y @octopusdeploy/mcp-server --help
   ```

## Planned Features

- **Octopus CLI Integration**: Support for write operations using the `octopus` CLI to create, modify, and delete resources on Octopus Server
- **OCL Skill**: Work with Git-versioned deployment processes and runbooks using Octopus Configuration Language

## Contributing

Contributions are welcome! To add new skills or improve existing ones:

1. Fork the repository
2. Create a feature branch
3. Add or modify skills in the `skills/` directory
4. Follow the existing skill patterns (SKILL.md + supporting docs)
5. Update the README
6. Submit a pull request

### Plugin Structure

```
octopus-deploy-devops/
├── .mcp.json                          # MCP server configuration
├── .claude-plugin/
│   └── plugin.json                    # Plugin metadata
└── skills/
    ├── deployment-investigation/      # Operations: Troubleshooting
    ├── infrastructure-audit/          # Operations: Infrastructure health
    ├── kubernetes-operations/         # Operations: Kubernetes (2025.3+)
    ├── release-management/            # Operations: Release workflows
    ├── space-environment-navigation/  # Operations: Instance exploration
    └── tenant-management/             # Operations: Multi-tenancy
```

### Skill Pattern

Each skill follows this structure:

```
skills/skill-name/
├── SKILL.md                   # Skill definition with YAML frontmatter
├── guide-1.md                 # Supporting documentation
└── guide-2.md                 # Additional documentation
```

## Resources

- [Octopus Deploy Documentation](https://octopus.com/docs)
- [Octopus Deploy MCP Server](https://github.com/OctopusDeploy/mcp-server)

## License

This plugin is maintained by Octopus Deploy.

## Support

For issues or questions:
- Plugin issues: [GitHub Issues](https://github.com/OctopusDeploy/claude-code-plugins-public/issues)
- Octopus Deploy support: [Octopus Support](https://octopus.com/support)
- MCP server issues: [MCP Server Issues](https://github.com/OctopusDeploy/mcp-server/issues)
