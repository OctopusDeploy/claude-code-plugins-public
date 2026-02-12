# Octopus Deploy Plugin

Claude Code plugin for Octopus Deploy operations and workflows.

## Development Guidelines

### Scripts

**Language:** All scripts use PowerShell Core 7+ for cross-platform compatibility (Windows, macOS, Linux), native Windows support, and robust structured data handling.

**Requirements:**
- Generic parameters (no hardcoded project/environment/space IDs)
- Environment variables for credentials (`$env:OCTOPUS_API_KEY`, `$env:OCTOPUS_SERVER_URL`)
- Clear error messages and parameter validation

### Skills

Skills must be generic and reusable across any Octopus Deploy instance.

**Structure:** Each skill includes:
- `SKILL.md` - Frontmatter and documentation
- `scripts/` - PowerShell scripts

### Testing

Test scripts on multiple platforms (Windows, macOS, Linux) and Octopus versions. Verify error handling and parameter validation.

### Documentation

Keep documentation concise, generic (use example values like `Projects-123`, not real production IDs), example-driven, and up-to-date.
