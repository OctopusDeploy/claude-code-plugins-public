# Octopus Deploy REST API and Automation

This document details the Octopus Deploy REST API, CLI, and automation capabilities.

## REST API Overview

Octopus is built API-first: every action in the UI is available via the API.

### API Characteristics

- **Hypermedia-Driven**: Links guide navigation (HATEOAS)
- **JSON Format**: All requests and responses
- **RESTful Design**: Standard HTTP methods
- **Space-Aware**: Resources scoped to spaces

### API Base URL

```
https://your-octopus-server/api
```

**Space-Scoped Endpoints:**
```
https://your-octopus-server/api/{spaceId}/projects
```

**Default Space:**
If a default space is configured, space ID can be omitted.

## Authentication

### API Keys

**Creating API Key:**
1. Navigate to User Profile
2. Select "API Keys"
3. Create new key with description
4. Store securely (shown only once)

**Using API Key:**
```
X-Octopus-ApiKey: API-XXXXXXXXXXXXXXXXXXXXXXXXXX
```

Or as query parameter:
```
?apiKey=API-XXXXXXXXXXXXXXXXXXXXXXXXXX
```

### Service Accounts

For automation, create dedicated service accounts:
- Clear ownership/purpose
- Minimal required permissions
- Rotatable credentials

## Common API Operations

### Projects

**List Projects:**
```http
GET /api/{spaceId}/projects
```

**Get Project:**
```http
GET /api/{spaceId}/projects/{projectId}
```

**Get Deployment Process:**
```http
GET /api/{spaceId}/projects/{projectId}/deploymentprocesses
```

### Releases

**Create Release:**
```http
POST /api/{spaceId}/releases
{
  "ProjectId": "Projects-1",
  "ChannelId": "Channels-1",
  "Version": "1.2.3",
  "SelectedPackages": [
    {
      "ActionName": "Deploy Package",
      "PackageReferenceName": "",
      "Version": "1.2.3"
    }
  ]
}
```

**List Releases:**
```http
GET /api/{spaceId}/projects/{projectId}/releases
```

### Deployments

**Create Deployment:**
```http
POST /api/{spaceId}/deployments
{
  "ReleaseId": "Releases-1",
  "EnvironmentId": "Environments-1",
  "TenantId": "Tenants-1", // optional
  "Comments": "Deployed via API"
}
```

**Get Deployment:**
```http
GET /api/{spaceId}/deployments/{deploymentId}
```

### Environments

**List Environments:**
```http
GET /api/{spaceId}/environments
```

**Get Environment:**
```http
GET /api/{spaceId}/environments/{environmentId}
```

### Deployment Targets

**List Targets:**
```http
GET /api/{spaceId}/machines
```

**Get Target:**
```http
GET /api/{spaceId}/machines/{machineId}
```

**Discover Target:**
```http
POST /api/{spaceId}/machines/discover
{
  "Host": "target.example.com",
  "Port": 10933,
  "Type": "TentaclePassive"
}
```

### Runbooks

**List Runbooks:**
```http
GET /api/{spaceId}/projects/{projectId}/runbooks
```

**Run Runbook:**
```http
POST /api/{spaceId}/runbookRuns
{
  "RunbookId": "Runbooks-1",
  "RunbookSnapshotId": "RunbookSnapshots-1",
  "EnvironmentId": "Environments-1"
}
```

### Tasks

**Get Task:**
```http
GET /api/tasks/{taskId}
```

**Get Task Details:**
```http
GET /api/tasks/{taskId}/details
```

**Cancel Task:**
```http
POST /api/tasks/{taskId}/cancel
```

### Variables

**Get Variables:**
```http
GET /api/{spaceId}/variables/{variableSetId}
```

**Update Variables:**
```http
PUT /api/{spaceId}/variables/{variableSetId}
{
  "Variables": [...]
}
```

## Octopus CLI

Command-line interface for common operations.

### Installation

**Windows:**
```powershell
choco install octopus-cli
```

**macOS:**
```bash
brew install octopuscli
```

**Linux:**
```bash
# Download from GitHub releases
```

**Docker:**
```bash
docker pull octopusdeploy/octo
```

### Configuration

**Environment Variables:**
```bash
export OCTOPUS_CLI_SERVER=https://your-octopus-server
export OCTOPUS_CLI_API_KEY=API-XXXXXXXXXX
export OCTOPUS_SPACE=Default
```

### Common Commands

**Create Release:**
```bash
octo create-release \
  --project "MyProject" \
  --version "1.2.3" \
  --packageVersion "1.2.3"
```

**Deploy Release:**
```bash
octo deploy-release \
  --project "MyProject" \
  --releaseNumber "1.2.3" \
  --deployTo "Production"
```

**Push Package:**
```bash
octo push \
  --package "MyApp.1.2.3.zip" \
  --replace-existing
```

**List Environments:**
```bash
octo list-environments
```

**Run Runbook:**
```bash
octo run-runbook \
  --project "MyProject" \
  --runbook "Backup Database" \
  --environment "Production"
```

## Octopus.Client (.NET SDK)

Strongly-typed .NET library for API access.

### Installation

```powershell
Install-Package Octopus.Client
```

### Basic Usage

```csharp
using Octopus.Client;
using Octopus.Client.Model;

var endpoint = new OctopusServerEndpoint("https://octopus-server", "API-KEY");
var repository = new OctopusRepository(endpoint);

// Get a project
var project = repository.Projects.FindByName("MyProject");

// Create a release
var release = new ReleaseResource
{
    ProjectId = project.Id,
    Version = "1.2.3"
};
repository.Releases.Create(release);

// Deploy
var deployment = new DeploymentResource
{
    ReleaseId = release.Id,
    EnvironmentId = "Environments-1"
};
repository.Deployments.Create(deployment);
```

### Space-Scoped Operations

```csharp
var space = repository.Spaces.FindByName("MySpace");
var spaceRepository = repository.ForSpace(space);

var project = spaceRepository.Projects.FindByName("MyProject");
```

## Build Server Integration

### GitHub Actions

```yaml
- name: Install Octopus CLI
  uses: OctopusDeploy/install-octopus-cli-action@v1

- name: Create Release
  uses: OctopusDeploy/create-release-action@v1
  with:
    api_key: ${{ secrets.OCTOPUS_API_KEY }}
    server: ${{ secrets.OCTOPUS_SERVER }}
    project: "MyProject"
    release_number: ${{ github.run_number }}

- name: Deploy Release
  uses: OctopusDeploy/deploy-release-action@v1
  with:
    api_key: ${{ secrets.OCTOPUS_API_KEY }}
    server: ${{ secrets.OCTOPUS_SERVER }}
    project: "MyProject"
    release_number: ${{ github.run_number }}
    environments: "Development"
```

### Azure DevOps

```yaml
- task: OctopusPush@4
  inputs:
    OctoConnectedServiceName: 'Octopus Connection'
    Package: '$(Build.ArtifactStagingDirectory)/*.nupkg'

- task: OctopusCreateRelease@4
  inputs:
    OctoConnectedServiceName: 'Octopus Connection'
    ProjectName: 'MyProject'
    ReleaseNumber: '$(Build.BuildNumber)'

- task: OctopusDeployRelease@4
  inputs:
    OctoConnectedServiceName: 'Octopus Connection'
    ProjectName: 'MyProject'
    ReleaseNumber: '$(Build.BuildNumber)'
    Environments: 'Development'
```

### Jenkins

```groovy
pipeline {
    stages {
        stage('Deploy to Octopus') {
            steps {
                octopusCreateRelease(
                    project: 'MyProject',
                    releaseVersion: "${BUILD_NUMBER}",
                    serverId: 'octopus-server'
                )
                octopusDeployRelease(
                    project: 'MyProject',
                    releaseVersion: "${BUILD_NUMBER}",
                    environment: 'Development',
                    serverId: 'octopus-server'
                )
            }
        }
    }
}
```

### TeamCity

Use the Octopus Deploy plugin:
- OctopusDeploy: Create Release
- OctopusDeploy: Deploy Release
- OctopusDeploy: Push Package

## Automation Patterns

### Infrastructure as Code

**Create Environments Programmatically:**
```powershell
$environments = @("Dev", "Test", "Staging", "Production")

foreach ($env in $environments) {
    $body = @{
        Name = $env
        Description = "$env environment"
        AllowDynamicInfrastructure = $true
    } | ConvertTo-Json

    Invoke-RestMethod `
        -Uri "$OctopusUrl/api/Spaces-1/environments" `
        -Method Post `
        -Headers @{"X-Octopus-ApiKey" = $ApiKey} `
        -Body $body `
        -ContentType "application/json"
}
```

### Tenant Provisioning

```powershell
function New-OctopusTenant {
    param(
        [string]$Name,
        [string]$ProjectId,
        [string[]]$EnvironmentIds
    )

    # Create tenant
    $tenant = @{
        Name = $Name
        ProjectEnvironments = @{
            $ProjectId = $EnvironmentIds
        }
    } | ConvertTo-Json -Depth 3

    $result = Invoke-RestMethod `
        -Uri "$OctopusUrl/api/Spaces-1/tenants" `
        -Method Post `
        -Headers @{"X-Octopus-ApiKey" = $ApiKey} `
        -Body $tenant `
        -ContentType "application/json"

    return $result
}
```

### Deployment Pipeline

```powershell
# Create release
$release = octo create-release `
    --project "MyProject" `
    --version $version `
    --outputFormat json | ConvertFrom-Json

# Deploy to each environment
$environments = @("Development", "Test", "Staging", "Production")

foreach ($env in $environments) {
    octo deploy-release `
        --project "MyProject" `
        --releaseNumber $release.Version `
        --deployTo $env `
        --waitForDeployment

    # Run smoke tests
    # ...
}
```

### Bulk Operations

```powershell
# Update all targets with specific tag
$targets = (Invoke-RestMethod `
    -Uri "$OctopusUrl/api/Spaces-1/machines?roles=web-server" `
    -Headers @{"X-Octopus-ApiKey" = $ApiKey}).Items

foreach ($target in $targets) {
    $target.MachinePolicyId = "MachinePolicies-2"

    Invoke-RestMethod `
        -Uri "$OctopusUrl/api/Spaces-1/machines/$($target.Id)" `
        -Method Put `
        -Headers @{"X-Octopus-ApiKey" = $ApiKey} `
        -Body ($target | ConvertTo-Json -Depth 10) `
        -ContentType "application/json"
}
```

## Webhooks and Subscriptions

### Subscription Events

Octopus can notify external systems of events:

**Event Types:**
- Deployment started/completed/failed
- Release created
- Machine health changed
- Runbook run completed

**Configuration:**
1. Navigate to Configuration → Subscriptions
2. Create subscription
3. Select event types
4. Configure webhook URL

### Webhook Payload

```json
{
  "Timestamp": "2024-01-15T10:30:00Z",
  "EventType": "DeploymentSucceeded",
  "Payload": {
    "DeploymentId": "Deployments-1",
    "ProjectId": "Projects-1",
    "EnvironmentId": "Environments-1",
    "ReleaseId": "Releases-1"
  }
}
```

## Best Practices

### API Usage
- Use service accounts for automation
- Implement retry logic for transient failures
- Respect rate limits
- Cache frequently accessed data

### Security
- Store API keys securely
- Use minimal required permissions
- Rotate credentials regularly
- Audit API access

### Error Handling
- Check response status codes
- Parse error messages
- Implement appropriate retries
- Log failures for investigation

### Performance
- Use bulk operations when possible
- Implement pagination for large result sets
- Cache configuration data
- Parallelize independent operations
