# Kubernetes Troubleshooting

## Understanding Kubernetes Resources in Octopus

Octopus Deploy can manage various Kubernetes resources:
- Deployments
- StatefulSets
- DaemonSets
- Services
- ConfigMaps
- Secrets
- Ingresses

## Pod Status Investigation

### 1. Check Pod Health

To investigate pod health:

1. Get live status for the deployment
2. Review pod states (Running, Pending, Failed, Crashed)
3. Check container statuses
4. Review restart counts

**Pod States:**
- `Running`: Pod is healthy and running
- `Pending`: Pod is waiting to be scheduled or start
- `Failed`: Pod terminated with error
- `Succeeded`: Pod completed successfully
- `Unknown`: Pod state cannot be determined

### 2. Container Status Analysis

Each pod can have multiple containers:

**Container States:**
- `Running`: Container is executing
- `Waiting`: Container is waiting to start
- `Terminated`: Container has stopped

**Waiting Reasons:**
- `ContainerCreating`: Container is being created
- `CrashLoopBackOff`: Container keeps crashing
- `ImagePullBackOff`: Cannot pull container image
- `CreateContainerError`: Error creating container

### 3. Pod Events and Errors

Review pod events for troubleshooting:

1. Get live status with events
2. Review event messages
3. Identify error patterns
4. Check timestamps for recent issues

**Common Events:**
- Image pull errors
- Resource constraints
- Scheduling failures
- Liveness/readiness probe failures

## Common Kubernetes Issues

### ImagePullBackOff

Cannot pull container image:

**Causes:**
- Image doesn't exist
- Incorrect image name/tag
- Registry authentication failure
- Network connectivity issues
- Rate limiting

**Investigation:**
- Verify image name and tag
- Check registry credentials
- Test registry connectivity
- Review pull secrets

### CrashLoopBackOff

Container keeps crashing and restarting:

**Causes:**
- Application startup failure
- Missing configuration
- Resource constraints
- Invalid command/arguments
- Dependency unavailable

**Investigation:**
- Check container logs
- Review restart count
- Verify resource requests/limits
- Validate configuration
- Check dependencies

### Pending Pods

Pods stuck in Pending state:

**Causes:**
- Insufficient cluster resources
- Node selector constraints
- Pod affinity/anti-affinity rules
- Resource quotas exceeded
- PersistentVolumeClaim issues

**Investigation:**
- Check pod events
- Review resource requests
- Verify node capacity
- Check node selectors
- Validate PVC status

### Failed Liveness/Readiness Probes

Health checks failing:

**Causes:**
- Application not responding
- Incorrect probe configuration
- Startup taking too long
- Resource constraints
- Network issues

**Investigation:**
- Review probe configuration
- Check application health endpoint
- Verify probe timeout/period
- Test endpoint manually
- Review application logs

## Deployment Troubleshooting

### Rollout Status

Check deployment rollout progress:

1. Get deployment live status
2. Review replica counts (desired vs ready vs available)
3. Check rollout strategy
4. Identify stuck rollouts

**Rollout Strategies:**
- `RollingUpdate`: Gradual replacement of pods
- `Recreate`: Stop all pods, then create new ones

**Stuck Rollouts:**
- New pods failing to start
- Old pods not terminating
- Resource constraints
- ImagePullBackOff on new version

### Replica Counts

Understanding replica status:

**Desired Replicas:**
- Target number of pod replicas
- Defined in deployment spec

**Current Replicas:**
- Total number of pods (old + new)
- During rollout, may exceed desired

**Ready Replicas:**
- Number of pods passing readiness checks
- Available to serve traffic

**Available Replicas:**
- Number of pods available for minimum time
- Considered stable

## Resource Issues

### Insufficient Resources

Pods can't start due to resource constraints:

**CPU Constraints:**
- Node has insufficient CPU
- CPU requests too high
- CPU limits too restrictive

**Memory Constraints:**
- Node has insufficient memory
- Memory requests too high
- OOMKilled (Out of Memory)

**Investigation:**
- Check pod resource requests/limits
- Review node capacity
- Identify resource-heavy workloads
- Consider cluster scaling

### Resource Limits

Containers hitting resource limits:

**CPU Throttling:**
- Container CPU usage capped at limit
- Performance degradation
- May need higher CPU limit

**Memory Limits:**
- Container terminated when exceeding memory limit
- OOMKilled status
- Need higher memory limit or fix memory leak

## Networking Issues

### Service Discovery

Pods can't connect to services:

**Causes:**
- Service not created
- Incorrect service selector
- DNS issues
- Network policy blocking traffic
- Service endpoints not ready

**Investigation:**
- Verify service exists
- Check service selector matches pod labels
- Test DNS resolution
- Review network policies
- Check service endpoints

### Ingress Problems

External traffic not reaching services:

**Causes:**
- Ingress not configured
- Incorrect ingress rules
- Ingress controller issues
- SSL/TLS certificate problems
- DNS not pointing to ingress

**Investigation:**
- Verify ingress configuration
- Check ingress controller logs
- Test ingress endpoints
- Validate certificates
- Verify DNS records

## Configuration Issues

### ConfigMaps and Secrets

Configuration not loading correctly:

**Causes:**
- ConfigMap/Secret doesn't exist
- Incorrect volume mount
- Wrong keys referenced
- Missing permissions

**Investigation:**
- Verify ConfigMap/Secret exists
- Check volume mount paths
- Validate key names
- Review RBAC permissions

### Environment Variables

Environment variables not set correctly:

**Causes:**
- Typo in variable name
- Value not defined
- ConfigMap/Secret reference incorrect
- Special characters not escaped

**Investigation:**
- Review deployment spec
- Check environment variable sources
- Validate ConfigMap/Secret references
- Test with known good values

## Monitoring Best Practices

### Regular Health Checks
- Monitor pod status continuously
- Check deployment rollout progress
- Review resource utilization
- Track restart counts

### Proactive Monitoring
- Set up alerts for pod failures
- Monitor resource usage trends
- Track deployment success rates
- Review event logs regularly

### Documentation
- Document common issues and solutions
- Maintain runbooks for troubleshooting
- Track configuration changes
- Keep deployment history

## Common Troubleshooting Questions

- "Why are my pods not starting?"
- "What's causing the CrashLoopBackOff?"
- "Why can't the container pull the image?"
- "Are all replicas healthy and ready?"
- "What errors are in the pod events?"
- "Why is the deployment rollout stuck?"
- "How many times has this pod restarted?"
- "What's the status of containers in this pod?"
