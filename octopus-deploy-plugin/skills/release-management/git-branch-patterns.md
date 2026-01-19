# Git Branch Deployment Patterns

Git-based deployments in Octopus Deploy allow you to deploy directly from Git branches.

## Available from Octopus Server 2021.2+

The `get_branches` tool requires Octopus Server version 2021.2 or later.

## Common Git Branch Workflows

### 1. Feature Branch Deployments

Deploy feature branches for testing:

1. List available Git branches
2. Create a release from a feature branch
3. Deploy to development environment
4. Test and validate changes

**Use Cases:**
- Testing feature branches before merge
- Parallel feature development
- Preview deployments

### 2. Main/Master Branch Releases

Standard production deployment workflow:

1. Merge feature branches to main
2. Create release from main branch
3. Deploy through environment progression
4. Track production deployment

**Pattern:**
```
feature/* → main → dev → staging → production
```

### 3. Hotfix Branch Deployments

Emergency fixes bypassing normal workflow:

1. Create hotfix branch from production tag
2. Deploy hotfix to production quickly
3. Merge back to main after deployment

**Pattern:**
```
production → hotfix/* → production
hotfix/* → main (after deployment)
```

### 4. Release Branch Strategy

Long-lived release branches for versions:

1. Create release branch from main
2. Apply patches and fixes to release branch
3. Deploy release branch to production
4. Maintain multiple release branches simultaneously

**Pattern:**
```
main → release/1.0 → production
main → release/2.0 → production
```

## Querying Git Branches

### List All Branches
"Show me all Git branches for the WebApp project"

### Find Specific Branch
"Does a branch named 'feature/new-api' exist?"

### Recent Branches
"What are the most recently created branches?"

### Branch Naming Patterns
Common patterns:
- `feature/*` - Feature development
- `hotfix/*` - Emergency fixes
- `release/*` - Release branches
- `bugfix/*` - Bug fixes
- `main` or `master` - Primary branch

## Git-Based Release Creation

### From Feature Branch
Create a release to test feature branch changes:
1. Select feature branch
2. Create release with branch name
3. Deploy to dev environment
4. Test changes in isolation

### From Main Branch
Standard release creation:
1. Ensure features merged to main
2. Create release from main
3. Progress through environments
4. Tag successful production deployment

## Best Practices

### Branch Protection
- Protect main/master branches
- Require pull request reviews
- Enforce status checks before merge

### Branch Lifecycle
- Delete branches after merge
- Keep branch naming consistent
- Limit long-lived branches

### Release Tagging
- Tag production releases
- Use semantic versioning for tags
- Link releases to Git commits

### Deployment Process
- Deploy feature branches only to dev
- Require main branch for production
- Use branch filters in deployment conditions

## Integration with Release Management

Combining Git branches with release management:

1. **Branch Discovery**: Query available branches
2. **Release Creation**: Create releases from specific branches
3. **Deployment Tracking**: Track which branch version is deployed where
4. **Rollback**: Deploy previous branch commits for rollback

## Common Questions

- "Which Git branch is currently deployed in production?"
- "What branches are available for the API project?"
- "Can I deploy the feature/new-dashboard branch to dev?"
- "What's the difference between the main branch and the release/2.0 branch?"
- "Which commits are in the latest release?"
