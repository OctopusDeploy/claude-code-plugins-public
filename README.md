# Octopus Deploy Claude Code Plugins

## Installation

### 1. Add Marketplace and Install Plugin

```bash
/plugin marketplace add https://github.com/OctopusDeploy/claude-code-plugins-public
/plugin install octopus-deploy-devops
```

### 2. Configure Environment Variables

The plugin requires these environment variables to be configured:

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
