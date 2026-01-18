# ArgoCD Sync Waves

## What are Sync Waves?

Sync waves control the **order** in which resources are applied during sync.

## Key Differences: Hooks vs Waves

| Feature | Hooks | Waves |
|---------|-------|-------|
| Purpose | Lifecycle operations | Resource ordering |
| Resource Types | Usually Jobs | Any K8s resource |
| Execution | At specific phases | During Sync phase |
| Lifecycle | PreSync, Sync, PostSync | Only during Sync |
| Typical Use | Migrations, tests | Dependencies, ordering |

## How Waves Work

### Wave Annotation
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"
```

### Wave Number
- **Default**: 0 (if no annotation)
- **Range**: Any integer (-∞ to +∞)
- **Order**: Lower number = earlier deployment
- **Negative values**: Deploy before wave 0
- **Positive values**: Deploy after wave 0

### Execution Flow
```
Wave -5 resources
    ↓ (wait for healthy)
Wave -2 resources  
    ↓ (wait for healthy)
Wave 0 resources (default)
    ↓ (wait for healthy)
Wave 1 resources
    ↓ (wait for healthy)
Wave 5 resources
    ↓
Complete
```

## Example: Typical Wave Strategy

```yaml
# Wave -3: Namespace (if needed)
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  annotations:
    argocd.argoproj.io/sync-wave: "-3"

---
# Wave -2: ConfigMaps and Secrets
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  annotations:
    argocd.argoproj.io/sync-wave: "-2"

---
# Wave -1: PersistentVolumeClaims
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
  annotations:
    argocd.argoproj.io/sync-wave: "-1"

---
# Wave 0: Database (StatefulSets)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  annotations:
    argocd.argoproj.io/sync-wave: "0"

---
# Wave 1: Backend API (depends on database)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  annotations:
    argocd.argoproj.io/sync-wave: "1"

---
# Wave 2: Frontend (depends on backend)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  annotations:
    argocd.argoproj.io/sync-wave: "2"

---
# Wave 3: Ingress (last to deploy)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    argocd.argoproj.io/sync-wave: "3"
```

## Common Wave Patterns

### Pattern 1: Infrastructure First
```
Wave -5: Namespaces, ResourceQuotas, LimitRanges
Wave -4: RBAC (ServiceAccounts, Roles, RoleBindings)
Wave -3: Secrets, ConfigMaps
Wave -2: PVCs, StorageClasses
Wave -1: Custom Resource Definitions (CRDs)
Wave  0: Operators, Controllers
Wave  1: Database StatefulSets
Wave  2: Application Deployments
Wave  3: Services
Wave  4: Ingress
```

### Pattern 2: Layered Architecture
```
Wave 0: Data Layer (Databases)
Wave 1: Cache Layer (Redis, Memcached)
Wave 2: Backend Services
Wave 3: API Gateway
Wave 4: Frontend
Wave 5: Edge Services (CDN, Ingress)
```

### Pattern 3: Microservices Dependencies
```
Wave 0: Auth Service (others depend on it)
Wave 1: User Service
Wave 2: Product Service  
Wave 3: Order Service (depends on User + Product)
Wave 4: Notification Service
Wave 5: Frontend (depends on all)
```

## Combining Hooks and Waves

### Hook Waves
Hooks can also have wave annotations to order within their phase:
```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/sync-wave: "1"
```

### Complete Execution Order
```
PreSync Hooks:
  Wave -2
  Wave -1
  Wave 0
  Wave 1
    ↓
Sync Phase:
  Wave -2
  Wave -1
  Wave 0 (default)
  Wave 1
  Wave 2
    ↓
PostSync Hooks:
  Wave -1
  Wave 0
  Wave 1
```

## Health Checks and Waves

### Wait for Health
ArgoCD waits for all resources in a wave to be **healthy** before proceeding to next wave.

**Health Criteria:**
- Deployments: All replicas ready
- StatefulSets: All replicas ready
- Jobs: Completed successfully
- Pods: Running and ready
- Services: Endpoints available

### Skip Health Check
Can skip health check for specific resources:
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
```

## Advanced Wave Techniques

### Technique 1: Progressive Rollout
```yaml
# Deploy to different regions in waves
---
# Wave 1: Region US-East
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-us-east
  annotations:
    argocd.argoproj.io/sync-wave: "1"
---
# Wave 2: Region US-West
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-us-west
  annotations:
    argocd.argoproj.io/sync-wave: "2"
---
# Wave 3: Region EU
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-eu
  annotations:
    argocd.argoproj.io/sync-wave: "3"
```

### Technique 2: Canary with Waves
```yaml
# Wave 1: Canary (10%)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  replicas: 1
---
# Wave 2: Main (90%)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-main
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  replicas: 9
```

### Technique 3: Dependency Management
```yaml
# Service A (no dependencies)
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"
---
# Service B (depends on A)
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
---
# Service C (depends on A and B)
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "2"
```

## Real-World Example: E-Commerce Platform

```yaml
# Wave -3: Infrastructure
---
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce
  annotations:
    argocd.argoproj.io/sync-wave: "-3"

---
# Wave -2: Configuration
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: ecommerce
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: ecommerce
  annotations:
    argocd.argoproj.io/sync-wave: "-2"

---
# Wave -1: Storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: ecommerce
  annotations:
    argocd.argoproj.io/sync-wave: "-1"

---
# Wave 0: Database
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: ecommerce
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  serviceName: postgres
  replicas: 1
  # ... spec

---
# Wave 1: Cache Layer
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: ecommerce
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  replicas: 1
  # ... spec

---
# Wave 2: Auth Service (critical dependency)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: ecommerce
  annotations:
    argocd.argoproj.io/sync-wave: "2"

---
# Wave 3: Core Services
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: ecommerce
  annotations:
    argocd.argoproj.io/sync-wave: "3"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: ecommerce
  annotations:
    argocd.argoproj.io/sync-wave: "3"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inventory-service
  namespace: ecommerce
  annotations:
    argocd.argoproj.io/sync-wave: "3"

---
# Wave 4: Dependent Services
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: ecommerce
  annotations:
    argocd.argoproj.io/sync-wave: "4"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: ecommerce
  annotations:
    argocd.argoproj.io/sync-wave: "4"

---
# Wave 5: Frontend
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: ecommerce
  annotations:
    argocd.argoproj.io/sync-wave: "5"

---
# Wave 6: Ingress (last)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: ecommerce
  annotations:
    argocd.argoproj.io/sync-wave: "6"
```

## Troubleshooting Waves

### Issue 1: Wave Stuck
**Symptom:** Sync stuck waiting for wave to complete
```bash
# Check which resources are unhealthy
argocd app get <app-name>

# Check specific resource
kubectl describe <resource-type> <name> -n <namespace>

# Check events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

**Common Causes:**
- Image pull errors
- Resource limits exceeded
- Dependency not actually ready
- Health check misconfigured

### Issue 2: Wrong Order
**Symptom:** Resources deploying in wrong order
```bash
# Verify wave annotations
kubectl get <resource> -o yaml | grep sync-wave

# Check in ArgoCD
argocd app get <app> -o json | jq '.status.resources[] | {name, syncWave}'
```

### Issue 3: Waves Too Granular
**Symptom:** Sync takes too long
**Solution:** Combine resources that can deploy in parallel
```yaml
# Instead of:
service-a: wave 1
service-b: wave 2
service-c: wave 3

# Use:
service-a, service-b, service-c: wave 1  # Parallel
```

## Best Practices

### 1. Use Negative Waves for Infrastructure
```yaml
# Always deploy first
Namespace: wave -3
Secrets/ConfigMaps: wave -2
PVCs: wave -1
```

### 2. Group Resources Logically
```yaml
# Same wave for resources that can deploy in parallel
database-1: wave 0
database-2: wave 0
database-3: wave 0
```

### 3. Leave Gaps Between Waves
```yaml
# Bad:
layer-1: wave 0
layer-2: wave 1
layer-3: wave 2

# Good (allows inserting between):
layer-1: wave 0
layer-2: wave 10
layer-3: wave 20
```

### 4. Document Wave Strategy
```yaml
# Add comments explaining why
metadata:
  annotations:
    # Must deploy before backend (dependency)
    argocd.argoproj.io/sync-wave: "1"
```

### 5. Test Wave Order
Create test app with echo containers to verify order without side effects.

### 6. Monitor Wave Progress
```bash
# Watch sync progress
argocd app get <app> --watch

# See which wave is currently deploying
```

## Waves vs Alternatives

### Alternative 1: Separate Applications
**Waves:**
- Single application, ordered deployment
- Simpler to manage
- All or nothing sync

**Separate Apps:**
- Multiple applications
- Can sync independently
- More complex management

### Alternative 2: Dependencies in Code
**Waves:**
- Declarative in manifests
- ArgoCD handles ordering
- No code changes needed

**Init Containers:**
- Embedded in pod spec
- Runtime dependency checking
- Works in any deployment tool

### When to Use What?

**Use Waves When:**
- Resources in same application have dependencies
- Want declarative ordering
- Need to ensure infrastructure before apps

**Use Separate Apps When:**
- Resources belong to different teams
- Need independent sync control
- Want different sync policies

**Use Init Containers When:**
- Runtime dependency checking needed
- Non-Kubernetes dependencies (external APIs)
- Not using ArgoCD

## Complete Example: Multi-Tier Application

```yaml
# TIER 0: Foundation
---
kind: Namespace
metadata:
  annotations: {argocd.argoproj.io/sync-wave: "-10"}
---
kind: ResourceQuota
metadata:
  annotations: {argocd.argoproj.io/sync-wave: "-10"}

# TIER 1: Configuration
---
kind: ConfigMap
metadata:
  annotations: {argocd.argoproj.io/sync-wave: "-5"}
---
kind: Secret
metadata:
  annotations: {argocd.argoproj.io/sync-wave: "-5"}

# TIER 2: Storage
---
kind: PersistentVolumeClaim
metadata:
  annotations: {argocd.argoproj.io/sync-wave: "0"}

# TIER 3: Data Layer
---
kind: StatefulSet  # PostgreSQL
metadata:
  annotations: {argocd.argoproj.io/sync-wave: "10"}
---
kind: StatefulSet  # MongoDB
metadata:
  annotations: {argocd.argoproj.io/sync-wave: "10"}

# TIER 4: Cache
---
kind: Deployment  # Redis
metadata:
  annotations: {argocd.argoproj.io/sync-wave: "20"}

# TIER 5: Backend Services
---
kind: Deployment  # Auth Service
metadata:
  annotations: {argocd.argoproj.io/sync-wave: "30"}
---
kind: Deployment  # API Gateway
metadata:
  annotations: {argocd.argoproj.io/sync-wave: "40"}
---
kind: Deployment  # Microservices
metadata:
  annotations: {argocd.argoproj.io/sync-wave: "50"}

# TIER 6: Frontend
---
kind: Deployment
metadata:
  annotations: {argocd.argoproj.io/sync-wave: "60"}

# TIER 7: Ingress
---
kind: Ingress
metadata:
  annotations: {argocd.argoproj.io/sync-wave: "70"}
```

This creates clear separation and allows future insertions!
