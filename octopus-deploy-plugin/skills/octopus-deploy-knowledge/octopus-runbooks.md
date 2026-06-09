# Octopus Deploy Runbooks

This document details runbooks in Octopus Deploy for automating operations tasks.

## What are Runbooks?

Runbooks automate routine maintenance and emergency operations tasks that aren't application deployments. They provide the same benefits as deployments (consistency, audit trail, permissions) for day-2 operations.

## Runbooks vs Deployments

| Aspect | Deployments | Runbooks |
|--------|-------------|----------|
| Purpose | Deploy application releases | Operations tasks |
| Release Required | Yes | No |
| Lifecycles | Apply | Do not apply |
| Dashboard | Deployment dashboard | Runbook dashboard |
| Versioning | Releases (immutable) | Snapshots (can use draft) |
| Per Project | One deployment process | Multiple runbooks |

## Runbook Types

### Routine Operations
Replace manual operations and ClickOps:
- Database backups
- Log rotation
- Certificate renewals
- Cache clearing
- Health checks

### Emergency Operations
Reduce stress during incidents:
- Server restarts
- Service restarts
- DNS failover
- Database failover
- Scale operations

### Infrastructure Provisioning
Manage infrastructure lifecycle:
- Terraform apply/destroy
- CloudFormation deployments
- VM provisioning
- Container orchestration
- Environment setup

## Runbook Structure

### Runbook Process
Similar to deployment process with steps and actions:
- Script steps
- Package deployment (for tools/scripts)
- Manual interventions
- Health checks

### Runbook Variables
Runbooks share project variables with deployments:
- Connection strings
- Credentials
- Configuration values
- Certificates

**Scoping:**
Variables can be scoped to:
- Specific runbooks
- Deployment process only
- Both runbooks and deployments

## Snapshots vs Releases

### Snapshots (Non-Config-as-Code)

**Draft Snapshots:**
- Always use current process and variables
- Good for development and testing
- Not recommended for production

**Published Snapshots:**
- Captures process and variables at publish time
- Recommended for production operations
- Can have multiple published snapshots

### Config-as-Code Runbooks

Use git commits instead of snapshots:
- Run from specific branch/commit
- Same versioning as code
- No explicit publishing required

## Running Runbooks

### Manual Execution
1. Navigate to Project → Runbooks
2. Select runbook
3. Choose environment
4. Optionally select specific targets
5. Run Now

### Scheduled Triggers
Configure automatic execution:
- Cron-based schedules
- Specific days/times
- Multiple schedules per runbook

**Examples:**
- Daily backups at 2 AM
- Weekly maintenance Sunday nights
- Monthly reports on 1st of month

### Prompted Variables
Request input when running:
- Target database name
- Backup location
- Scale count
- Confirmation text

## Environment Selection

### Run Settings
Configure which environments a runbook can execute in:
- **All environments**: No restrictions
- **Specific environments**: Explicit list
- **Project lifecycle**: Environments from project's lifecycle

### Environment-Specific Behavior
Variables and steps can be scoped to environments:
- Different backup locations per environment
- Longer maintenance windows in production
- Skip certain steps in development

## Permissions

Runbooks have separate permissions from deployments:

**Runbook Permissions:**
- `RunbookView`: View runbook definition
- `RunbookEdit`: Modify runbook process
- `RunbookRunCreate`: Execute runbooks
- `RunbookRunView`: View run history

**Permission Patterns:**
- Developers can run in Dev
- DBAs can run database runbooks
- Ops team can run in all environments

## Retention Policies

### Run Retention
Control how long runbook run history is kept:
- Keep X runs per environment
- Default: 100 runs per environment

### Config-as-Code Considerations
When branches are deleted, retention defaults to 60-day time-based policy.

## Runbook Examples

### Database Backup
```
Steps:
1. Stop dependent services
2. Run backup script
3. Verify backup file
4. Upload to storage
5. Restart services
6. Send notification
```

### Certificate Renewal
```
Steps:
1. Request new certificate (Let's Encrypt)
2. Validate certificate
3. Install certificate
4. Update bindings
5. Restart web server
6. Verify HTTPS
```

### Emergency Restart
```
Steps:
1. Manual intervention (confirm restart)
2. Notify stakeholders
3. Graceful service stop
4. Clear caches
5. Start services
6. Health check
7. Update status page
```

### Infrastructure Provisioning
```
Steps:
1. Run Terraform plan
2. Manual intervention (review plan)
3. Run Terraform apply
4. Register new targets
5. Deploy baseline configuration
6. Health check
```

## Best Practices

### Design
- Keep runbooks focused (single responsibility)
- Use meaningful names
- Document purpose and prerequisites
- Include verification steps

### Safety
- Use manual interventions for destructive operations
- Implement dry-run modes where possible
- Test in non-production first
- Log all actions for audit

### Variables
- Reuse project variables
- Use prompted variables for dynamic input
- Scope appropriately

### Scheduling
- Choose appropriate times (maintenance windows)
- Consider dependencies
- Monitor scheduled run results
- Handle failures gracefully

### Permissions
- Principle of least privilege
- Separate by environment sensitivity
- Audit permission assignments

## Integration Patterns

### CI/CD Pipeline Integration
Trigger runbooks from build servers:
- Post-deployment verification
- Environment preparation
- Test data setup

### Monitoring Integration
Trigger runbooks from alerts:
- Auto-remediation
- Scaling responses
- Incident response

### ChatOps
Trigger runbooks from chat:
- Self-service operations
- Quick incident response
- Scheduled maintenance

## Runbook Triggers

### Scheduled Triggers
Configure in Runbook → Triggers:
- Select runbook
- Define schedule (cron or simple)
- Choose environments
- Set timezone

### Manual Triggers
Always available:
- From Octopus UI
- Via REST API
- Through Octopus CLI

### API Triggers
Integrate with external systems:
```
POST /api/{spaceId}/runbookRuns
{
  "RunbookId": "Runbooks-1",
  "EnvironmentId": "Environments-1"
}
```
