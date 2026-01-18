# Sync Waves Observation Guide

## Application: waves-demo

### Wave Architecture

```
Wave -3: Namespace
         ↓ (wait for healthy)
Wave -2: ConfigMaps, Secrets
         ↓ (wait for healthy)
Wave -1: PersistentVolumeClaims
         ↓ (wait for healthy)
Wave  0: PostgreSQL Database
         ↓ (wait for healthy)
Wave  1: Redis Cache
         ↓ (wait for healthy)
Wave  2: Auth Service (critical dependency)
         ↓ (wait for healthy)
Wave  3: User Service, Product Service (parallel)
         ↓ (wait for healthy)
Wave  4: API Gateway
         ↓ (wait for healthy)
Wave  5: Frontend
         ↓ (wait for healthy)
Wave  6: Ingress
         ↓
      Complete!
```

## Observation Instructions

### Terminal 1: Sync and Watch
```bash
# Start sync
argocd app sync waves-demo

# Watch sync progress
argocd app get waves-demo --watch
```

### Terminal 2: Watch Resources
```bash
# Watch all resources being created
watch 'kubectl get all,pvc,ingress -n waves-demo --sort-by=.metadata.creationTimestamp'
```

### Terminal 3: Monitor Events
```bash
# Watch events in real-time
kubectl get events -n waves-demo --sort-by='.lastTimestamp' --watch
```

## Expected Behavior

### Phase 1: Infrastructure (Waves -3 to -1)
**Timeline: 0-30 seconds**

You should see (in order):
1. Namespace created
2. ConfigMap and Secret created
3. PVC created

```bash
# Verify
kubectl get namespace waves-demo
kubectl get configmap,secret -n waves-demo
kubectl get pvc -n waves-demo
```

### Phase 2: Data Layer (Wave 0)
**Timeline: 30 seconds - 2 minutes**

PostgreSQL StatefulSet and Service:
```bash
# Watch postgres pod starting
kubectl get pods -n waves-demo -l app=postgres -w

# Wait for readiness probe to pass
# This can take 30-60 seconds
```

**ArgoCD waits here** until postgres is healthy!

### Phase 3: Cache Layer (Wave 1)
**Timeline: 2-3 minutes**

Only after postgres is healthy:
```bash
# Redis deployment starts
kubectl get deployment redis -n waves-demo -w

# Watch pod
kubectl get pods -n waves-demo -l app=redis
```

### Phase 4: Auth Service (Wave 2)
**Timeline: 3-4 minutes**

Critical dependency deploys:
```bash
kubectl get deployment auth-service -n waves-demo -w
```

### Phase 5: Backend Services (Wave 3)
**Timeline: 4-5 minutes**

**Note: These deploy in PARALLEL!**
```bash
# Both start at the same time
kubectl get deployment user-service product-service -n waves-demo -w
```

### Phase 6: API Gateway (Wave 4)
**Timeline: 5-6 minutes**

```bash
kubectl get deployment api-gateway -n waves-demo -w
```

### Phase 7: Frontend (Wave 5)
**Timeline: 6-7 minutes**

```bash
kubectl get deployment frontend -n waves-demo -w
```

### Phase 8: Ingress (Wave 6)
**Timeline: 7-8 minutes**

```bash
kubectl get ingress -n waves-demo
```

## Verification

### Check Wave Order in ArgoCD
```bash
# List all resources with their wave
argocd app get waves-demo -o json | jq -r '.status.resources[] | "\(.syncWave // 0)\t\(.kind)\t\(.name)"' | sort -n
```

### Verify All Resources Healthy
```bash
# All should show Synced and Healthy
argocd app get waves-demo

# Check in Kubernetes
kubectl get all -n waves-demo
```

### Access the Application
```bash
# Frontend via NodePort
curl http://localhost:30001

# Or port-forward
kubectl port-forward -n waves-demo svc/frontend 8081:80 &
curl http://localhost:8081
```

## Experiments

### Experiment 1: Observe Wave Blocking

```bash
# Delete the application
argocd app delete waves-demo --cascade

# Break the database intentionally
cd ../argocd-demo-apps/waves-demo/manifests
# Edit complete-app.yaml
# Change postgres image to invalid:latest
git add . && git commit -m "Break postgres" && git push

# Sync again
kubectl apply -f ../../argo-mastery/argocd/labs/app-definitions/waves-demo-app.yaml
argocd app sync waves-demo

# Observe: Sync STOPS at wave 0 (database)
# Waves 1-6 never deploy because wave 0 is unhealthy
```

**Key Learning:** Waves provide safety - if critical infrastructure fails, apps don't deploy.

### Experiment 2: Change Wave Order

```bash
# What if frontend deployed before backend?
# Edit manifests to swap waves
# Frontend: wave 3
# Backend services: wave 5

# Sync and observe
# Frontend pods will start but likely fail health checks
# because backend isn't available yet
```

**Key Learning:** Wave order matters for dependencies!

### Experiment 3: Remove Waves

```bash
# Remove all sync-wave annotations
# Everything deploys at once (wave 0)
# Observe: Less predictable order
# May work, may not, depending on timing
```

**Key Learning:** Waves provide deterministic deployment order.

### Experiment 4: Monitor Resource Usage During Waves

```bash
# Watch resource consumption
kubectl top pods -n waves-demo --watch

# Notice: Resources consumed gradually as waves deploy
# Not all at once
```

**Key Learning:** Waves can help with resource management.

## Troubleshooting

### Sync Stuck at Specific Wave

```bash
# Find which resources are unhealthy
argocd app get waves-demo | grep -A5 "Progressing\|Degraded"

# Check specific resource
kubectl describe <resource-type> <name> -n waves-demo

# Common issues:
# - Image pull errors
# - Resource limits exceeded
# - Readiness probe failing
# - Missing dependencies
```

### Skip Problematic Wave (Not Recommended)

```bash
# If absolutely necessary, can delete problematic resource
# This allows next wave to proceed
# BUT: This breaks intended architecture!
kubectl delete <resource> -n waves-demo
```

## Cleanup

```bash
# Delete application
argocd app delete waves-demo --cascade

# Or just the namespace
kubectl delete namespace waves-demo

# Fix any broken manifests
cd ../argocd-demo-apps
git revert HEAD  # if you broke something
git push
```

## Architecture Benefits Demonstrated

✅ **Ordered Deployment:** Infrastructure before applications
✅ **Dependency Management:** Services wait for dependencies
✅ **Parallel Deployment:** Wave 3 services deploy together
✅ **Safety:** Failed infrastructure blocks app deployment
✅ **Visibility:** Clear progression through tiers
✅ **Rollback:** Can rollback entire stack in reverse order

## Key Takeaways

1. **Waves control deployment order** - Lower numbers first
2. **Health checks between waves** - Must be healthy to proceed
3. **Parallel within wave** - Same wave number = parallel
4. **Blocking on failure** - Unhealthy wave blocks next waves
5. **Use negative waves for infra** - Config before apps
6. **Document wave strategy** - Comment why each wave number
7. **Test wave order** - Verify dependencies are correct
