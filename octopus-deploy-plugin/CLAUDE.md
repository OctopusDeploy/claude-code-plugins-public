# Octopus Deploy Plugin - Claude Instructions

## Scripts

**Use PowerShell Core 7+** for all scripts (cross-platform compatible).

**Never hardcode:** Project IDs, Environment IDs, Space IDs, or Server URLs. Use parameters and environment variables.

**Template:**
```powershell
param(
    [Parameter(Mandatory=$true)][string]$RequiredParam,
    [string]$OptionalParam = "default",
    [string]$ServerUrl = $env:OCTOPUS_SERVER_URL
)

if (-not $env:OCTOPUS_API_KEY) {
    Write-Host "❌ OCTOPUS_API_KEY not set" -ForegroundColor Red; exit 1
}

try {
    # Script logic
} catch {
    Write-Host "❌ Error: $($_.Exception.Message)" -ForegroundColor Red; throw
}
```
