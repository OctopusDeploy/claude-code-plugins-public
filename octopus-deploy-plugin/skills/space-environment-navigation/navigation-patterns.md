# Navigation and Exploration Patterns

## Getting Started with a New Instance

### 1. Identify Available Spaces

First step when exploring an Octopus instance:

1. List all spaces
2. Review space names and purposes
3. Identify which spaces you have access to
4. Understand space organization

**Common Space Patterns:**
- Environment-based: `Production Space`, `Development Space`
- Team-based: `Platform Team`, `Product Team`
- Service-based: `E-commerce`, `Internal Tools`

### 2. Verify User Permissions

Check your access level:

1. Get current user information
2. Review user permissions and teams
3. Identify admin vs user access
4. Understand access restrictions

**Questions to Answer:**
- What's my user role?
- Do I have space-level or instance-level permissions?
- Which teams am I part of?
- Can I perform administrative actions?

### 3. Explore Space Structure

For each space you need to work with:

1. List environments in the space
2. List projects in the space
3. Map project-environment relationships
4. Understand deployment paths

**Standard Environment Progression:**
```
Development → Test → Staging → Production
```

**Alternative Patterns:**
```
Dev → UAT → Production
Feature → Integration → Release → Production
```

## Multi-Space Instances

### Understanding Space Isolation

Spaces provide organizational isolation:

**Space Benefits:**
- Separate projects, environments, and configurations
- Team-based access control
- Independent deployment processes
- Isolated variable sets and resources

**When to Use Multiple Spaces:**
- Large organizations with multiple teams
- Separate product lines or services
- Development vs production isolation
- Client-specific deployments (MSP scenarios)

### Navigating Across Spaces

Cross-space navigation workflow:

1. List all spaces
2. Select target space
3. Query space-specific resources (projects, environments)
4. Switch contexts as needed

**Important:** Most resources are space-scoped. Always specify the space when querying projects, environments, releases, or deployments.

## Environment Organization

### Standard Environment Types

**Development Environments:**
- Used for active development and testing
- Frequently updated with latest changes
- May have relaxed security controls

**Staging/UAT Environments:**
- Pre-production validation
- Production-like configuration
- User acceptance testing
- Performance testing

**Production Environments:**
- Live customer-facing deployments
- Strict change control
- High availability requirements
- Monitoring and alerting

### Environment Lifecycle

Understanding environment progression:

1. **Early Stages**: Dev, Feature branches
2. **Testing Stages**: Test, Integration, QA
3. **Pre-Production**: Staging, UAT, Pre-prod
4. **Production**: Prod, Live

## Project Discovery

### Finding Projects in a Space

To explore available projects:

1. List all projects in the space
2. Review project names and descriptions
3. Identify project groups
4. Understand project purposes

**Common Project Patterns:**
- Service-based: `API`, `WebApp`, `Worker`
- Component-based: `Frontend`, `Backend`, `Database`
- Environment-based: `Prod-Infrastructure`, `Dev-Services`

### Project Grouping

Projects can be organized into groups:
- Logical grouping of related projects
- Shared deployment processes
- Common variable sets
- Team ownership

## Common Navigation Questions

### "Where am I?"
1. Get current user to verify authentication
2. List spaces to see instance organization
3. List projects to see available deployments

### "What can I deploy?"
1. List projects in the current space
2. Review project deployment processes
3. Check environment availability
4. Verify user permissions

### "How is this organized?"
1. List spaces for top-level organization
2. List environments in each space
3. Map projects to environments
4. Understand deployment paths

### "What's my access level?"
1. Get current user information
2. Review permissions and teams
3. Test access to different spaces
4. Identify any restrictions

## Navigation Best Practices

### Start Broad, Then Narrow
1. Begin with spaces (highest level)
2. Move to environments (deployment targets)
3. Explore projects (what gets deployed)
4. Deep dive into specific resources

### Understand Context
- Always know which space you're working in
- Verify environment names before deployments
- Check project configurations before releases
- Confirm user permissions for operations

### Map Relationships
- Projects deploy to environments
- Environments exist within spaces
- Users have space-specific permissions
- Resources are space-scoped (except spaces themselves)

## Example Exploration Workflow

Complete instance exploration:

```
1. "What spaces exist?"
   → List spaces

2. "What environments are in the Production space?"
   → List environments in Production space

3. "What projects deploy to Production?"
   → List projects in Production space

4. "Who am I and what can I access?"
   → Get current user

5. "What's deployed to Production environment?"
   → List deployments for Production environment
```

## Quick Reference

### Hierarchy
```
Instance
├── Spaces
│   ├── Environments
│   ├── Projects
│   ├── Releases
│   └── Deployments
└── Users (instance-level or space-scoped)
```

### Common Commands
- "List all spaces"
- "Show environments in [Space Name]"
- "List projects in [Space Name]"
- "What's my user information?"
- "What spaces can I access?"
