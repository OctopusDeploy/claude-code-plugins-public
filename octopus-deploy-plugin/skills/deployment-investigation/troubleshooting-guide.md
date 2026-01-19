# Deployment Troubleshooting Guide

## Common Investigation Patterns

### 1. Find Recent Failed Deployments

To identify recent deployment failures:

1. List recent deployments for a project
2. Filter by `TaskState = "Failed"`
3. Review the deployment details including task ID
4. Analyze the error messages

**Workflow:**
- Start by listing deployments in a specific space
- Narrow down by project if needed
- Get task details for failed deployments
- Review error logs and stack traces

### 2. Compare Failed vs Successful Deployments

When a deployment fails, compare it with the last successful deployment:

1. List recent deployments for the project
2. Identify the last successful deployment before the failure
3. Compare deployment processes and variables
4. Look for differences in packages, configuration, or environment

**Key Questions:**
- What changed between the successful and failed deployment?
- Are there environment-specific differences?
- Were there package version changes?

### 3. Analyze Task Error Messages

For deep error analysis:

1. Get task details with `get_task_details`
2. Review the full task log with `get_task_raw`
3. Extract error messages and stack traces
4. Cross-reference with environment configuration

**Look For:**
- Exception types and error codes
- Missing dependencies or configuration
- Timeout issues
- Network connectivity problems
- Permission errors

### 4. Environment-Specific Issues

When deployments work in some environments but not others:

1. List all environments in the space
2. Compare deployment history across environments
3. Review environment-specific variables
4. Check deployment target health in each environment

**Common Causes:**
- Environment-specific configuration differences
- Target availability or health issues
- Network connectivity variations
- Different package versions

### 5. Recurring Deployment Failures

For repeated failures of the same deployment:

1. Review deployment history for the project
2. Identify patterns in failure times or conditions
3. Check for infrastructure issues
4. Review recent changes to the deployment process

**Investigate:**
- Time-based patterns (failures at specific times)
- Target-specific issues
- Resource constraints
- Intermittent network issues
