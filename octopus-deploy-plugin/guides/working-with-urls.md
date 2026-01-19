# Working with Octopus Deploy URLs

This guide explains how to work with Octopus Deploy URLs when investigating deployments, tasks, and other resources.

## URL Structure Patterns

### Deployment URLs

**Format**: `https://{server}/app#/Spaces-{id}/projects/{slug}/deployments/releases/{version}/deployments/Deployments-{id}`

**Example**: `https://{your-octopus-server}/app#/Spaces-1/projects/my-project/deployments/releases/1.0.0/deployments/Deployments-123`

**Contains**:
- Space ID: `Spaces-1`
- Project slug: `my-project`
- Release version: `1.0.0`
- Deployment ID: `Deployments-123`
- **Does NOT contain task ID**

### Task URLs

**Format**: `https://{server}/app#/Spaces-{id}/tasks/ServerTasks-{id}`

**Example**: `https://{your-octopus-server}/app#/Spaces-1/tasks/ServerTasks-456`

**Contains**:
- Space ID: `Spaces-1`
- Task ID: `ServerTasks-456`

### Release URLs

**Format**: `https://{server}/app#/Spaces-{id}/projects/{slug}/deployments/releases/{version}`

**Example**: `https://{your-octopus-server}/app#/Spaces-1/projects/my-project/deployments/releases/1.0.0`

**Contains**:
- Space ID: `Spaces-1`
- Project slug: `my-project`
- Release version: `1.0.0`

### Project URLs

**Format**: `https://{server}/app#/Spaces-{id}/projects/{slug}`

**Example**: `https://{your-octopus-server}/app#/Spaces-1/projects/my-project`

**Contains**:
- Space ID: `Spaces-1`
- Project slug: `my-project`

## Resource Relationships

Understanding the relationships between Octopus resources is critical for working with URLs correctly.

### Deployment → Task

- Every deployment creates a **server task** to execute the deployment
- The deployment object contains a `TaskId` field that references the task
- **Critical**: Deployment URLs do NOT contain the task ID
- To get task details from a deployment URL, you must:
  1. Extract the deployment ID from the URL
  2. Query the deployment using `list_deployments`
  3. Get the `TaskId` from the deployment response
  4. Query the task using `get_task_details` or `get_task_by_id`

### Space ID → Space Name

- URLs contain space **IDs** (e.g., `Spaces-1`)
- MCP tools require space **names** (e.g., `"Team A Space"`)
- To resolve a space ID to a name:
  1. Use `list_spaces` to get all spaces
  2. Match the space ID from the URL
  3. Use the space name in tool calls

## URL Extraction Workflows

### Scenario 1: User provides a deployment URL and wants task logs

**URL**: `https://{your-octopus-server}/app#/Spaces-1/projects/my-project/deployments/Deployments-123`

**Workflow**:
1. Extract space ID: `Spaces-1`
2. Extract deployment ID: `Deployments-123`
3. Call `list_spaces` to find space name for `Spaces-1`
4. Call `list_deployments` with filters or directly query to get deployment details
5. Extract `TaskId` from deployment response (e.g., `ServerTasks-456`)
6. Call `get_task_details` with space name and task ID
7. Analyze task logs

**Common Mistake**: Trying to extract a task ID directly from the deployment URL (it doesn't exist there)

### Scenario 2: User provides a task URL

**URL**: `https://{your-octopus-server}/app#/Spaces-1/tasks/ServerTasks-456`

**Workflow**:
1. Extract space ID: `Spaces-1`
2. Extract task ID: `ServerTasks-456`
3. Call `list_spaces` to find space name for `Spaces-1`
4. Call `get_task_details` with space name and task ID
5. Analyze task logs

### Scenario 3: User provides a project URL and wants recent deployments

**URL**: `https://{your-octopus-server}/app#/Spaces-1/projects/my-project`

**Workflow**:
1. Extract space ID: `Spaces-1`
2. Extract project slug: `my-project`
3. Call `list_spaces` to find space name
4. Call `list_projects` to find project ID from slug
5. Call `list_deployments` filtered by project ID

## Common Pitfalls and How to Avoid Them

### Pitfall 1: Assuming deployment URLs contain task IDs

**Wrong approach**:
```
URL: https://{your-octopus-server}/.../Deployments-123
→ Extract task ID from URL (DOESN'T EXIST)
→ Call get_task_details with guessed task ID
→ ERROR: Task not found
```

**Correct approach**:
```
URL: https://{your-octopus-server}/.../Deployments-123
→ Extract deployment ID: Deployments-123
→ Call list_deployments to get deployment details
→ Extract TaskId from response: ServerTasks-456
→ Call get_task_details with correct task ID
→ SUCCESS
```

### Pitfall 2: Using space IDs instead of space names

**Wrong approach**:
```
URL contains: Spaces-1
→ Call get_task_details(spaceName: "Spaces-1", ...)
→ ERROR: Space not found
```

**Correct approach**:
```
URL contains: Spaces-1
→ Call list_spaces
→ Find space with Id "Spaces-1"
→ Extract Name: "Team A Space"
→ Call get_task_details(spaceName: "Team A Space", ...)
→ SUCCESS
```

### Pitfall 3: Guessing or hallucinating IDs

**Wrong approach**:
```
URL: https://{your-octopus-server}/.../Deployments-123
→ Guess that task ID might be ServerTasks-999
→ Call get_task_details with guessed ID
→ ERROR: Task not found (guessed wrong ID)
```

**Correct approach**:
```
URL: https://{your-octopus-server}/.../Deployments-123
→ Only extract IDs that are actually present in the URL
→ For IDs not in the URL, query the API to get them
→ Never guess or assume ID values
```

## Quick Reference Table

| URL Type | Contains | Requires API Call | To Get |
|----------|----------|-------------------|---------|
| Deployment | Deployment ID, Space ID | ✅ | Task ID |
| Task | Task ID, Space ID | ✅ | Space Name |
| Release | Release Version, Space ID, Project Slug | ✅ | Project ID, Space Name |
| Project | Project Slug, Space ID | ✅ | Project ID, Space Name |

## URL Pattern Regex

For reference, here are regex patterns for extracting IDs from URLs:

```regex
Space ID:      /Spaces-(\d+)/
Deployment ID: /Deployments-(\d+)/
Task ID:       /ServerTasks-(\d+)/
Project Slug:  /projects\/([^\/]+)/
Release Ver:   /releases\/([^\/]+)/
```

## Best Practices

1. **Always extract IDs explicitly from URLs** - Never guess or assume ID values
2. **Follow the resource chain** - Deployment → Task, Project → Deployments
3. **Resolve space names early** - Get space names before calling other tools
4. **Validate extracted IDs** - Check that IDs match expected patterns
5. **Handle URL variations** - Support both cloud and self-hosted URLs
6. **Provide clear error messages** - Explain what went wrong and how to fix it

## Examples from Real Usage

### Example 1: Successful deployment investigation

User: "Investigate this deployment: https://{your-octopus-server}/app#/Spaces-1/projects/my-project/deployments/Deployments-123"

Correct workflow:
1. Extract `Spaces-1` → Resolve to "Team A Space"
2. Extract `Deployments-123`
3. Call `list_deployments(spaceName: "Team A Space")` with filters
4. Find deployment, get `TaskId: "ServerTasks-456"`
5. Call `get_task_details(spaceName: "Team A Space", taskId: "ServerTasks-456")`
6. Analyze logs

### Example 2: Failed deployment investigation (incorrect approach)

User: "Investigate this deployment: https://{your-octopus-server}/app#/Spaces-1/projects/my-project/deployments/Deployments-123"

Incorrect workflow that led to errors:
1. Extract `Spaces-1` → Resolve to "Team A Space"
2. ❌ Guess task ID as "ServerTasks-999" (hallucinated, not from URL)
3. ❌ Call `get_task_details(spaceName: "Team A Space", taskId: "ServerTasks-999")`
4. ❌ ERROR: Task not found

Root cause: The task ID was not in the URL and was incorrectly guessed instead of being retrieved from the deployment object.
