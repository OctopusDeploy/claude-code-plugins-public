# Octopus Deploy Claude Code Plugin

An Octopus Deploy plugin for [Claude Code](https://claude.com/claude-code). It bundles the Octopus Deploy MCP server, an agent, and a set of skills so Claude can inspect, query, diagnose, and help configure your Octopus Deploy instance using live data.

## What's included

- **`octopus-deploy-agent`** — an agent that investigates the live state of your Octopus Server, traces configuration, and diagnoses deployment failures.
- **Skills:**
  - `octopus-deploy-knowledge` — the source of truth for how Octopus models projects, releases, lifecycles, environments, variables, runbooks, tenants, and spaces.
  - `octopus-onboarding` — guides a first end-to-end Octopus setup, from connecting a repo to a first real deployment.
  - `writing-ocl` — authoring Config-as-Code (`.ocl`) files.
  - `diagnose-deployment-failure` — structured investigation of failed deployments.
  - `octopus-audit-report` — generating audit reports from your instance.
- **MCP server** — the [Octopus Deploy MCP server](https://github.com/OctopusDeploy/mcp-server), preconfigured in `.mcp.json`.

## Installation

### 1. Add the marketplace and install the plugin

In Claude Code:

```bash
/plugin marketplace add OctopusDeploy/octopus-claude-plugins
/plugin install octopus-deploy-devops
```

### 2. Configure environment variables

The plugin's MCP server needs these environment variables:

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
