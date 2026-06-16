# Kubernetes Rolling Update - Command Reference 

## Table of Contents
- [Pre-Update Commands](#pre-update-commands)
- [Rolling Update Execution](#rolling-update-execution)
- [Monitoring & Status](#monitoring--status)
- [Verification Commands](#verification-commands)
- [Rollout Management](#rollout-management)
- [Debugging & Troubleshooting](#debugging--troubleshooting)
- [Advanced Commands](#advanced-commands)

---

## Pre-Update Commands

### Check Current Deployment Status

```bash
# View deployment details
kubectl get deployment nginx-deployment

# Detailed deployment information
kubectl describe deployment nginx-deployment

# Check current image version
kubectl describe deployment nginx-deployment | grep Image

# View deployment in YAML format
kubectl get deployment nginx-deployment -o yaml

# Get container names in deployment
kubectl get deployment nginx-deployment -o jsonpath='{.spec.template.spec.containers[*].name}'
```

### Check Current Pods

```bash
# List all pods for nginx deployment
kubectl get pods -l app=nginx

# Detailed pod information
kubectl get pods -l app=nginx -o wide

# Show pod status with node information
kubectl get pods -o wide

# Count running pods
kubectl get pods -l app=nginx --no-headers | wc -l
```

### Export Current Configuration (Backup)

```bash
# Backup deployment configuration
kubectl get deployment nginx-deployment -o yaml > nginx-deployment-backup.yaml

# Save current state
kubectl get deployment nginx-deployment -o json > nginx-deployment-backup.json
```

---

## Rolling Update Execution

### Primary Update Command

```bash
# Update deployment image (recommended method)
kubectl set image deployment/nginx-deployment nginx=nginx:1.17 --record
```

### Alternative Update Methods

```bash
# Using kubectl patch
kubectl patch deployment nginx-deployment -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.17"}]}}}}'

# Using kubectl apply (if you have the YAML file)
kubectl apply -f nginx-deployment.yaml

# Interactive edit
kubectl edit deployment nginx-deployment

# Update with annotation
kubectl annotate deployment nginx-deployment kubernetes.io/change-cause="Update to nginx:1.17"
```

### Update Multiple Containers

```bash
# If deployment has multiple containers
kubectl set image deployment/nginx-deployment nginx=nginx:1.17 sidecar=sidecar:v2 --record
```

---

## Monitoring & Status

### Watch Rollout Progress

```bash
# Monitor rollout status (blocks until complete)
kubectl rollout status deployment/nginx-deployment

# Watch rollout with timeout
kubectl rollout status deployment/nginx-deployment --timeout=5m

# Watch pods in real-time
kubectl get pods -l app=nginx -w

# Watch all resources
kubectl get all -w
```

### Real-Time Monitoring

```bash
# Continuous pod status updates
watch kubectl get pods -l app=nginx

# Monitor deployment events
kubectl get events --watch

# Watch specific namespace
kubectl get events -n default --watch
```

---

## Verification Commands

### Verify Update Success

```bash
# Check deployment status
kubectl get deployment nginx-deployment

# Verify new image is deployed
kubectl describe deployment nginx-deployment | grep Image

# Check pod image versions
kubectl get pods -l app=nginx -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'

# Verify all pods are ready
kubectl get pods -l app=nginx -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

### Check Pod Health

```bash
# Get pod status
kubectl get pods -l app=nginx

# Check pod readiness
kubectl get pods -l app=nginx -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# View pod details
kubectl describe pod <pod-name>

# Check container status
kubectl get pods -l app=nginx -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[*].ready}{"\n"}{end}'
```

### Validate Deployment Configuration

```bash
# Check replica count
kubectl get deployment nginx-deployment -o jsonpath='{.status.replicas}'

# Check available replicas
kubectl get deployment nginx-deployment -o jsonpath='{.status.availableReplicas}'

# Check updated replicas
kubectl get deployment nginx-deployment -o jsonpath='{.status.updatedReplicas}'

# Full status summary
kubectl get deployment nginx-deployment -o jsonpath='{.metadata.name}{"\t"}{.status.replicas}{"\t"}{.status.availableReplicas}{"\t"}{.status.updatedReplicas}{"\n"}'
```

---

## Rollout Management

### View Rollout History

```bash
# Show rollout history
kubectl rollout history deployment/nginx-deployment

# Show specific revision details
kubectl rollout history deployment/nginx-deployment --revision=2

# View all revisions with details
kubectl rollout history deployment/nginx-deployment --revision=0
```

### Pause and Resume Rollout

```bash
# Pause ongoing rollout
kubectl rollout pause deployment/nginx-deployment

# Resume paused rollout
kubectl rollout resume deployment/nginx-deployment

# Check if rollout is paused
kubectl get deployment nginx-deployment -o jsonpath='{.spec.paused}'
```

### Rollback Operations

```bash
# Rollback to previous version
kubectl rollout undo deployment/nginx-deployment

# Rollback to specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# Dry-run rollback (preview)
kubectl rollout undo deployment/nginx-deployment --dry-run=client

# Rollback and monitor
kubectl rollout undo deployment/nginx-deployment && kubectl rollout status deployment/nginx-deployment
```

### Restart Deployment

```bash
# Restart deployment (rolling restart)
kubectl rollout restart deployment/nginx-deployment
```

---

## Debugging & Troubleshooting

### View Logs

```bash
# View logs from a specific pod
kubectl logs <pod-name>

# Follow logs in real-time
kubectl logs -f <pod-name>

# View logs from previous container instance
kubectl logs <pod-name> --previous

# Logs from all pods with label
kubectl logs -l app=nginx --all-containers=true

# Tail last 50 lines
kubectl logs <pod-name> --tail=50

# Logs with timestamps
kubectl logs <pod-name> --timestamps
```

### Check Events

```bash
# View all events
kubectl get events

# Sort events by timestamp
kubectl get events --sort-by=.metadata.creationTimestamp

# Filter events for deployment
kubectl get events --field-selector involvedObject.name=nginx-deployment

# Watch events in real-time
kubectl get events --watch
```

### Pod Troubleshooting

```bash
# Execute command in pod
kubectl exec -it <pod-name> -- /bin/bash

# Check nginx configuration
kubectl exec <pod-name> -- nginx -t

# View nginx version
kubectl exec <pod-name> -- nginx -v

# Check pod resource usage
kubectl top pod <pod-name>

# Get pod YAML
kubectl get pod <pod-name> -o yaml

# Describe pod for detailed info
kubectl describe pod <pod-name>
```

### Network Troubleshooting

```bash
# Check service endpoints
kubectl get endpoints

# Describe service
kubectl describe service nginx-service

# Test connectivity from another pod
kubectl run -it --rm debug --image=busybox --restart=Never -- wget -O- http://nginx-service
```

### Resource Issues

```bash
# Check node resources
kubectl top nodes

# Check pod resource usage
kubectl top pods -l app=nginx

# View resource quotas
kubectl get resourcequota

# Check limit ranges
kubectl get limitrange
```

---

## Advanced Commands

### Scale Deployment

```bash
# Scale deployment
kubectl scale deployment nginx-deployment --replicas=5

# Autoscale deployment
kubectl autoscale deployment nginx-deployment --min=2 --max=10 --cpu-percent=80
```

### Update Strategy Configuration

```bash
# View current update strategy
kubectl get deployment nginx-deployment -o jsonpath='{.spec.strategy}'

# Set max surge and max unavailable
kubectl patch deployment nginx-deployment -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'
```

### Labels and Selectors

```bash
# Add label to deployment
kubectl label deployment nginx-deployment environment=production

# Update label
kubectl label deployment nginx-deployment version=1.17 --overwrite

# Remove label
kubectl label deployment nginx-deployment version-

# Get pods by label
kubectl get pods -l app=nginx,environment=production
```

### Annotations

```bash
# Add annotation
kubectl annotate deployment nginx-deployment description="Updated to nginx 1.17"

# View annotations
kubectl get deployment nginx-deployment -o jsonpath='{.metadata.annotations}'
```

### Export and Apply

```bash
# Export running configuration
kubectl get deployment nginx-deployment -o yaml --export > deployment.yaml

# Apply configuration from file
kubectl apply -f deployment.yaml

# Apply with record
kubectl apply -f deployment.yaml --record

# Validate without applying
kubectl apply -f deployment.yaml --dry-run=client
```

### Cleanup Commands

```bash
# Delete specific pod (will be recreated)
kubectl delete pod <pod-name>

# Delete all pods (will be recreated)
kubectl delete pods -l app=nginx

# Delete deployment (careful!)
kubectl delete deployment nginx-deployment

# Force delete stuck pod
kubectl delete pod <pod-name> --grace-period=0 --force
```

---

## Quick Reference Table

| Task | Command |
|------|---------|
| Update image | `kubectl set image deployment/nginx-deployment nginx=nginx:1.17 --record` |
| Check status | `kubectl rollout status deployment/nginx-deployment` |
| View history | `kubectl rollout history deployment/nginx-deployment` |
| Rollback | `kubectl rollout undo deployment/nginx-deployment` |
| Pause rollout | `kubectl rollout pause deployment/nginx-deployment` |
| Resume rollout | `kubectl rollout resume deployment/nginx-deployment` |
| View pods | `kubectl get pods -l app=nginx` |
| View logs | `kubectl logs -f <pod-name>` |
| Describe pod | `kubectl describe pod <pod-name>` |
| Scale | `kubectl scale deployment nginx-deployment --replicas=5` |

---

## Command Cheat Sheet

```bash
# Complete rolling update workflow
kubectl get deployment nginx-deployment                           # 1. Check current state
kubectl set image deployment/nginx-deployment nginx=nginx:1.17 --record  # 2. Update image
kubectl rollout status deployment/nginx-deployment                # 3. Monitor rollout
kubectl get pods -l app=nginx                                     # 4. Verify pods
kubectl describe deployment nginx-deployment | grep Image         # 5. Confirm image
kubectl rollout history deployment/nginx-deployment               # 6. Check history
```

---

**Note**: Replace `<pod-name>` with actual pod name and adjust label selectors as needed for your environment.
