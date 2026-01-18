# ArgoCD Sync Strategies and Options

## Manual vs Automated Sync

### Manual Sync (Default)
```yaml
syncPolicy: {}  # No automated sync
```

**Characteristics:**
- User must explicitly trigger sync
- Good for production (controlled deployments)
- Review changes before applying
- Can use `argocd app sync` or UI

### Automated Sync
```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

**Characteristics:**
- Syncs automatically when OutOfSync detected
- Good for development environments
- Faster feedback loop
- Less manual work

## Automated Sync Options

### prune
```yaml
automated:
  prune: true  # Delete resources not in Git
```

**What it does:**
- Removes K8s resources that exist in cluster but not in Git
- Ensures cluster only has what Git defines
- Prevents orphaned resources

**Use case:**
- Removed a service from Git → ArgoCD deletes it from cluster

**Warning:**
- Can delete resources unexpectedly
- Be careful in production

### selfHeal
```yaml
automated:
  selfHeal: true  # Revert manual changes
```

**What it does:**
- Reverts manual kubectl changes
- Enforces Git as source of truth
- Happens every reconciliation (3 min)

**Example:**
1. Git says 3 replicas
2. Someone does: `kubectl scale --replicas=5`
3. After 3 minutes: ArgoCD scales back to 3

**Use case:**
- Prevent configuration drift
- Enforce GitOps discipline

### allowEmpty
```yaml
automated:
  allowEmpty: false  # Don't sync if no resources
```

**What it does:**
- Prevents syncing when Git path has no resources
- Avoids accidental deletion of all resources

## Sync Options

### CreateNamespace
```yaml
syncOptions:
  - CreateNamespace=true
```

**What it does:**
- Auto-creates destination namespace if doesn't exist
- No need to pre-create namespaces

### PrunePropagationPolicy
```yaml
syncOptions:
  - PrunePropagationPolicy=foreground
```

**Options:**
- `foreground`: Wait for dependents to be deleted first
- `background`: Delete immediately, K8s garbage collector handles deps
- `orphan`: Don't delete dependents

**Use case:**
- Deleting Deployment should wait for Pods to terminate

### PruneLast
```yaml
syncOptions:
  - PruneLast=true
```

**What it does:**
- Prune resources AFTER new resources are healthy
- Prevents downtime during updates

**Example:**
1. New deployment created and healthy
2. THEN old deployment deleted
3. Zero downtime achieved

### ApplyOutOfSyncOnly
```yaml
syncOptions:
  - ApplyOutOfSyncOnly=true
```

**What it does:**
- Only applies resources that are actually different
- Skips resources already in sync
- Faster syncs, less K8s API calls

### ServerSideApply
```yaml
syncOptions:
  - ServerSideApply=true
```

**What it does:**
- Uses Kubernetes server-side apply (1.16+)
- Better handling of field ownership
- Handles CRDs better

**Advantage:**
- Multiple controllers can manage same resource
- Better conflict resolution

### RespectIgnoreDifferences
```yaml
syncOptions:
  - RespectIgnoreDifferences=true
```

**What it does:**
- Honors ignoreDifferences configuration
- Useful when combined with other tools

## ignoreDifferences

### Purpose
Tell ArgoCD to ignore certain fields when comparing

### Common Use Cases

#### 1. Ignore HPA-managed replicas
```yaml
ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
      - /spec/replicas
```

**Why:** HPA controls replicas, we don't want to conflict

#### 2. Ignore status fields
```yaml
ignoreDifferences:
  - group: "*"
    kind: "*"
    jqPathExpressions:
      - .status
```

**Why:** Status is runtime, not configuration

#### 3. Ignore annotations added by other tools
```yaml
ignoreDifferences:
  - group: apps
    kind: Deployment
    jqPathExpressions:
      - .metadata.annotations.["kubectl.kubernetes.io/last-applied-configuration"]
```

#### 4. Ignore fields managed by other controllers
```yaml
ignoreDifferences:
  - group: apps
    kind: Deployment
    managedFieldsManagers:
      - kube-controller-manager
```

## Retry Configuration

```yaml
syncPolicy:
  retry:
    limit: 5              # Max attempts
    backoff:
      duration: 5s        # Initial wait
      factor: 2           # Multiplier (5s, 10s, 20s, 40s, 80s)
      maxDuration: 3m     # Cap at 3 minutes
```

**When retries happen:**
- Sync fails (network error, invalid manifest, etc.)
- ArgoCD automatically retries with exponential backoff

**Example timeline:**
- Attempt 1: Fails
- Wait 5s
- Attempt 2: Fails
- Wait 10s (5s * 2)
- Attempt 3: Fails
- Wait 20s (10s * 2)
- Attempt 4: Fails
- Wait 40s (20s * 2)
- Attempt 5: Final attempt

## Health Assessment

### Default Health Checks

ArgoCD has built-in health for:
- Deployments: All replicas ready
- StatefulSets: All replicas ready
- Services: Endpoints exist
- Ingress: Has address
- PVCs: Bound to PV

### Custom Health Checks

```yaml
metadata:
  annotations:
    argocd.argoproj.io/health-check: |
      hs = {}
      if obj.status ~= nil and obj.status.readyReplicas == obj.spec.replicas then
        hs.status = "Healthy"
        hs.message = "All replicas ready"
        return hs
      end
      hs.status = "Progressing"
      return hs
```

**Health Status Values:**
- `Healthy`: Resource is functioning correctly
- `Progressing`: Resource is being created/updated
- `Degraded`: Resource exists but not functioning
- `Suspended`: Resource is suspended (e.g., CronJob)
- `Missing`: Resource should exist but doesn't
- `Unknown`: Cannot determine health

### Global Custom Health Checks

Can configure in `argocd-cm` ConfigMap:
```yaml
resource.customizations: |
  cert-manager.io/Certificate:
    health.lua: |
      hs = {}
      if obj.status ~= nil then
        if obj.status.conditions ~= nil then
          for i, condition in ipairs(obj.status.conditions) do
            if condition.type == "Ready" and condition.status == "False" then
              hs.status = "Degraded"
              hs.message = condition.message
              return hs
            end
            if condition.type == "Ready" and condition.status == "True" then
              hs.status = "Healthy"
              hs.message = condition.message
              return hs
            end
          end
        end
      end
      hs.status = "Progressing"
      hs.message = "Waiting for certificate"
      return hs
```

## Sync Strategies Summary

| Strategy | Use Case | Risk Level |
|----------|----------|------------|
| Manual | Production | Low |
| Auto (no prune/selfHeal) | Staging | Medium |
| Auto + selfHeal | Development | Medium |
| Auto + prune | Development | High |
| Auto + both | Dev/Test only | Highest |

## Best Practices

1. **Start with manual sync** - Move to automated gradually
2. **Use selfHeal in dev** - Enforce GitOps discipline
3. **Use prune carefully** - Can delete unexpected resources
4. **ignoreDifferences for HPA** - Avoid conflicts
5. **Implement retries** - Handle transient failures
6. **Custom health checks for CRDs** - Better status visibility
7. **Test in dev first** - Before applying to production
8. **Document exceptions** - Why ignoring certain differences

## Lab Exercises Complete
- ✅ Created app with custom health checks
- ✅ Tested automated sync
- ✅ Tested self-heal capability
- ✅ Tested prune functionality
- ✅ Configured ignoreDifferences
- ✅ Understood all sync options
