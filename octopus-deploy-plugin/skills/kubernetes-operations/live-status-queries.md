# Live Status Query Examples

## Basic Status Queries

### Check All Pods in Namespace
"Show me the status of all pods in the 'production' namespace"

### Check Specific Deployment
"What's the status of the 'api-deployment' deployment?"

### Check Pod Health
"Are all pods healthy in the 'web-app' deployment?"

### Check Service Status
"Show me the status of the 'frontend-service' service"

## Troubleshooting Queries

### Find Failing Pods
"Which pods are in Failed or Error state?"

### Check Restart Counts
"Show me pods with high restart counts"

### Investigate CrashLoopBackOff
"Why are pods in CrashLoopBackOff state?"

### Find ImagePullBackOff Issues
"Which pods can't pull their container images?"

### Check Pending Pods
"Show me all pods stuck in Pending state"

## Deployment Queries

### Rollout Status
"Is the deployment rollout complete?"

### Replica Status
"How many replicas are ready vs desired?"

### Check New Deployment
"Has the new version rolled out successfully?"

### Rollout Progress
"What's the progress of the current deployment rollout?"

## Resource Queries

### Pod Resource Usage
"Show me resource usage for pods in this deployment"

### Container Status
"What's the status of containers in pod X?"

### Check Resource Limits
"Are any pods hitting resource limits?"

### Node Capacity
"Is there enough capacity for this deployment?"

## Container Queries

### Container Health
"Are all containers in the pod healthy?"

### Container Restarts
"How many times has this container restarted?"

### Container State
"Why is the container in a Waiting state?"

### Container Logs
"Show me recent container events and errors"

## Service and Networking

### Service Endpoints
"Are service endpoints ready?"

### Check Service Discovery
"Can pods reach this service?"

### Ingress Status
"Is the ingress correctly configured?"

### Network Connectivity
"Are there any network-related issues?"

## Environment-Specific Queries

### Production Health
"Show me the health of all production Kubernetes deployments"

### Staging Validation
"Are all staging deployments healthy before promoting to production?"

### Development Status
"What's the status of development namespace pods?"

## Comparison Queries

### Compare Environments
"Compare pod health between dev and prod environments"

### Before/After Deployment
"What changed after the latest deployment?"

### Version Comparison
"Which pods are running the old vs new version?"

## Monitoring Queries

### Overall Health
"Give me an overview of Kubernetes cluster health"

### Recent Deployments
"Show me status of deployments from the last 24 hours"

### Failure Trends
"Are there any recurring pod failures?"

### Resource Trends
"Is resource usage increasing over time?"

## Advanced Queries

### Multi-Namespace Status
"Show me pod status across all namespaces"

### Resource Type Queries
"What's the status of all StatefulSets?"

### Label-Based Queries
"Show me all pods with label 'app=frontend'"

### Custom Resource Status
"What's the status of custom resources in this namespace?"

## Query Parameters

When querying live status, you can specify:

**Resource Type:**
- Pod
- Deployment
- StatefulSet
- DaemonSet
- Service
- Ingress
- ConfigMap
- Secret

**Namespace:**
- Specific namespace name
- All namespaces

**Resource Name:**
- Specific resource name
- Pattern matching

**Filters:**
- By labels
- By status
- By creation time
- By owner

## Example Query Patterns

### Pattern: "Show status of X"
- "Show status of api-deployment"
- "Show status of pods in production namespace"
- "Show status of frontend service"

### Pattern: "Why is X happening?"
- "Why are pods failing?"
- "Why can't the container start?"
- "Why is the deployment stuck?"

### Pattern: "Check X"
- "Check pod health"
- "Check deployment rollout"
- "Check service endpoints"

### Pattern: "Find X"
- "Find failing pods"
- "Find pods with high restarts"
- "Find unhealthy containers"

### Pattern: "Compare X"
- "Compare dev and prod deployment status"
- "Compare before and after deployment"
- "Compare replica counts"

## Real-World Scenarios

### Scenario 1: Deployment Failed
"The deployment just failed. What went wrong?"

**Investigation Steps:**
1. Check deployment status
2. Review pod events
3. Check container states
4. Identify error messages

### Scenario 2: Service Down
"Users can't access the application. What's the issue?"

**Investigation Steps:**
1. Check pod availability
2. Review service endpoints
3. Verify ingress configuration
4. Test connectivity

### Scenario 3: High Restart Count
"A pod keeps restarting. Why?"

**Investigation Steps:**
1. Check restart count
2. Review container status
3. Check resource limits
4. Review application logs

### Scenario 4: Slow Rollout
"The deployment is taking too long. What's happening?"

**Investigation Steps:**
1. Check rollout progress
2. Review replica counts
3. Identify stuck pods
4. Check resource availability

## Best Practices

### Query Frequency
- Production: Monitor continuously
- Staging: Check before/after deployments
- Development: Check on-demand

### Alert Triggers
- Pod restart count > threshold
- Deployment stuck > timeout
- Pods in Failed state
- Resource usage > limit

### Regular Checks
- Daily health checks
- Pre-deployment validation
- Post-deployment verification
- Weekly resource review
