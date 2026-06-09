<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://github.com/octopusdeploy/mcp-server/blob/main/images/OctopusDeploy_Logo_DarkMode.png?raw=true">
  <source media="(prefers-color-scheme: light)" srcset="https://github.com/octopusdeploy/mcp-server/blob/main/images/OctopusDeploy_Logo_LightMode.png?raw=true">
  <img alt="Octopus Deploy Logo" src="https://github.com/octopusdeploy/mcp-server/blob/main/images/OctopusDeploy_Logo_LightMode.png?raw=true" />
</picture>

# Claude Code Plugin for Octopus Deploy

[Octopus](https://octopus.com) makes it easy to deliver software to Kubernetes, multi-cloud, on-prem infrastructure, and anywhere else. Automate the release, deployment, and operations of your software and AI workloads with a tool that can handle CD at scale in ways no other tool can.

This plugin combines the power of the [Octopus MCP Server](https://github.com/OctopusDeploy/mcp-server) and a specialized set of skills to enable [Claude Code](https://claude.com/claude-code) to interact with Octopus Deploy in an automated fashion.

The MCP connects directly to your Octopus instance and can perform most actions that a regular user can:
- Create projects and deployment processes
- Create releases
- Execute runbooks
- Configure project variables
- Find any interruptions requiring human attention
- Inspect audit log
- And much more

The plugin comes with a set of skills to help with:
- Connecting a repository to Octopus
- Writing OCL ([Octopus Configuration Language](https://octopus.com/docs/projects/version-control/ocl-file-format))
- Diagnosing deployment failures and fixing them
- Producing an SOC2 audit report

## Getting Started

### 1. Add the marketplace and install the plugin

In Claude Code:

```bash
/plugin marketplace add OctopusDeploy/octopus-claude-plugins
/plugin install octopus-deploy-devops
```

### 2. Configure environment variables

The plugin's MCP server connects to your Octopus instance using these environment variables. We recommend setting up a separate [service account](https://octopus.com/docs/security/users-and-teams/service-accounts) with restricted permissions per Space.

**Unix/Mac (bash/zsh):**
```bash
export OCTOPUS_SERVER_URL="https://your-instance.octopus.app"
export OCTOPUS_API_KEY="API-XXXXXXXXXXXXX"
```

**Windows (PowerShell):**
```powershell
$env:OCTOPUS_SERVER_URL = "https://your-instance.octopus.app"
$env:OCTOPUS_API_KEY = "API-XXXXXXXXXXXXX"
```

Restart Claude Code after setting them so the MCP server picks them up.

## Examples

Here is an example of what you can do with this plugin:
- 📊 **Show me the deployment status of my project, and when it was last deployed to tenant X**
- 🔍 **Investigate and diagnose why a deployment has failed**
- 🔧 **Use the appropriate runbook to fix the failure**
- 🚀 **Redeploy to tenant**

## Resources

- [Octopus Deploy Documentation](https://octopus.com/docs)
- [Octopus Deploy MCP Server](https://github.com/OctopusDeploy/mcp-server)

## License

This plugin is maintained by Octopus Deploy.

## Support

For issues or questions:
- Plugin issues: [GitHub Issues](https://github.com/OctopusDeploy/octopus-claude-plugins/issues)
- Octopus Deploy support: [Octopus Support](https://octopus.com/support)
- MCP server issues: [MCP Server Issues](https://github.com/OctopusDeploy/mcp-server/issues)
