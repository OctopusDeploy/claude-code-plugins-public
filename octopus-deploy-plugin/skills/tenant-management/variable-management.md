# Tenant Variable Management

## Variable Types

### Common Variables

Variables shared across all projects connected to a tenant:

**Use Cases:**
- Tenant-wide settings
- Shared database connections
- Common API endpoints
- General configuration

**Example:**
```
Tenant.Region = "US-East"
Tenant.DataCenter = "AWS-Virginia"
Tenant.SupportEmail = "support@customer-abc.com"
```

### Project Variables

Variables specific to a single project:

**Use Cases:**
- Project-specific configuration
- Application settings
- Service endpoints
- Feature flags

**Example:**
```
WebApp.DatabaseConnection = "Server=db-abc.com;Database=webappdb"
API.ServiceUrl = "https://api.customer-abc.com"
API.RateLimitPerMinute = "1000"
```

### Environment Variables

Variables with environment-specific values:

**Use Cases:**
- Different values per environment
- Dev vs Production configuration
- Environment-specific URLs
- Deployment-specific settings

**Example:**
```
Development:
  API.Url = "https://dev-api.customer-abc.com"
Production:
  API.Url = "https://api.customer-abc.com"
```

## Variable Configuration

### Setting Up Tenant Variables

To configure tenant variables:

1. **Identify Required Variables**
   - Review project templates
   - Check deployment process requirements
   - List environment-specific needs

2. **Configure Common Variables**
   - Set tenant-wide settings
   - Configure shared resources
   - Define general parameters

3. **Configure Project Variables**
   - Set project-specific values
   - Override common variables if needed
   - Configure project templates

4. **Add Environment Overrides**
   - Set environment-specific values
   - Override project defaults
   - Configure deployment targets

### Variable Templates

Define reusable variable templates:

**Template Benefits:**
- Consistent variable structure
- Required vs optional variables
- Default values
- Validation rules

**Example Template:**
```
DatabaseConnection (Required)
ApiEndpoint (Required)
CustomDomain (Optional)
EnableFeatureX (Optional, Default: false)
```

## Managing Missing Variables

### Identifying Missing Variables

Use `get_missing_tenant_variables` to find:
- Required variables without values
- Variables needed for deployment
- Incomplete tenant configuration

**Common Missing Variables:**
- New variables added to templates
- Required fields not configured
- Environment-specific overrides missing

### Resolving Missing Variables

To fix missing variables:

1. **Identify Missing Variables**
   - Query tenant for missing values
   - Review template requirements
   - Check environment scoping

2. **Determine Correct Values**
   - Consult documentation
   - Contact tenant owner
   - Review similar tenant configurations

3. **Set Variable Values**
   - Configure in Octopus UI
   - Use Octopus API
   - Bulk update via script

4. **Validate Configuration**
   - Recheck for missing variables
   - Test deployment
   - Verify application behavior

## Variable Validation

### Pre-Deployment Validation

Before deploying to a tenant:

1. **Check for Missing Variables**
   - Query missing variables
   - Ensure all required values set
   - Validate environment-specific values

2. **Validate Variable Format**
   - Check connection strings
   - Verify URL formats
   - Validate credentials (without exposing)

3. **Test Configuration**
   - Validate against known good values
   - Check for common mistakes
   - Ensure compatibility

### Configuration Auditing

Regular variable audits:

**Audit Checks:**
- Are all required variables configured?
- Are sensitive variables properly secured?
- Are variable values consistent?
- Are there unused variables?

**Common Issues:**
- Outdated variable values
- Incorrect environment scoping
- Missing sensitive value protection
- Duplicate or conflicting variables

## Variable Security

### Securing Sensitive Variables

Protect sensitive values:

**Sensitive Variable Types:**
- Database passwords
- API keys
- Authentication tokens
- Encryption keys
- Certificate passwords

**Best Practices:**
- Mark variables as sensitive
- Use Azure Key Vault or AWS Secrets Manager
- Rotate credentials regularly
- Limit access to sensitive values
- Audit sensitive variable usage

### Access Control

Control who can view/edit variables:

**Permission Levels:**
- View variables (read-only)
- Edit variables (update values)
- Edit variable templates (structure)
- View sensitive values (restricted)

## Variable Troubleshooting

### Common Variable Issues

**Variable Not Found:**
- Variable name mismatch
- Incorrect scoping
- Variable not defined in template

**Wrong Variable Value:**
- Environment scoping incorrect
- Override not applied
- Variable cache issue

**Missing Required Variable:**
- New variable added to template
- Tenant not updated
- Environment not configured

### Debugging Variable Problems

To troubleshoot variable issues:

1. **Review Variable Configuration**
   - Get all tenant variables
   - Check variable scoping
   - Verify variable names

2. **Check Deployment Logs**
   - Look for variable resolution errors
   - Identify missing variables
   - Check variable substitution

3. **Compare with Working Tenant**
   - Review similar tenant configuration
   - Identify differences
   - Copy working values

4. **Validate Variable Templates**
   - Check template requirements
   - Verify variable definitions
   - Update tenant configuration

## Variable Management Best Practices

### Organization
- Use consistent naming conventions
- Group related variables
- Document variable purposes
- Maintain variable templates

### Maintenance
- Regularly audit variable values
- Remove unused variables
- Update outdated values
- Validate required variables

### Security
- Protect sensitive values
- Limit variable access
- Rotate credentials regularly
- Audit variable changes

### Documentation
- Document variable meanings
- Provide example values
- Explain scoping rules
- Maintain change history

## Common Variable Questions

- "What variables are configured for this tenant?"
- "Which variables are missing required values?"
- "What's the value of DatabaseConnection for this tenant?"
- "Are all required variables set for production deployment?"
- "Which project variables does this tenant have?"
- "What environment-specific overrides are configured?"
- "How do I bulk-update variables for multiple tenants?"
