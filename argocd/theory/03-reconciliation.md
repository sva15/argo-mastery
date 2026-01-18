# Reconciliation in ArgoCD

## What is Reconciliation?

**Reconciliation** is the process where ArgoCD:
1. Fetches desired state from Git
2. Fetches live state from Kubernetes
3. Compares them
4. Determines if action needed

## Reconciliation Loop

ArgoCD Application Controller runs reconciliation loop:
- **Default**: Every 3 minutes
- **Configurable**: Can be changed
- **Always running**: Continuous monitoring

## The Loop Process

```
┌─────────────────────────────────────────┐
│                                         │
│  1. Fetch from Git (Repository Server) │
│     - Get latest commit                 │
│     - Generate manifests                │
│                                         │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│                                         │
│  2. Fetch from K8s (Live State)        │
│     - Get actual resources              │
│     - Check health status               │
│                                         │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│                                         │
│  3. Compare (Diff)                      │
│     - Find differences                  │
│     - Determine sync status             │
│                                         │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│                                         │
│  4. Update Application Status           │
│     - OutOfSync / Synced                │
│     - Healthy / Progressing / Degraded  │
│                                         │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│                                         │
│  5. Take Action (if needed)             │
│     - Auto-sync if enabled              │
│     - Self-heal if enabled              │
│     - Or wait for manual sync           │
│                                         │
└──────────────┬──────────────────────────┘
               │
               ▼
         (Wait 3 minutes)
               │
               └──────────> (Back to Step 1)
```

## Reconciliation Timeout

**What it means:**
- Max time allowed for reconciliation operation
- Default: 180 seconds (3 minutes)
- If exceeded: Operation fails

**Why timeout?**
- Prevent hung operations
- Resource protection
- Fast failure detection

**Configure timeout:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  timeout.reconciliation: 180s  # 3 minutes default
```

## Git Webhook vs Polling

### Polling (Default)
- ArgoCD checks Git every 3 minutes
- **Advantage**: Simple, works everywhere
- **Disadvantage**: Up to 3 min delay

### Webhook (Recommended for production)
- Git notifies ArgoCD immediately on push
- **Advantage**: Instant notification (<1 second)
- **Disadvantage**: Requires network access to ArgoCD

### How Webhooks Work
```
Developer pushes to Git
         ↓
Git server sends HTTP POST to ArgoCD
         ↓
ArgoCD API Server receives webhook
         ↓
Triggers immediate reconciliation
         ↓
No need to wait 3 minutes!
```

## Refresh vs Sync

### Refresh
- **What**: Fetch latest from Git and compare
- **Does NOT** change cluster state
- **Use**: Check for drift without deploying
- **Triggers**: Automatic (every 3 min) or manual

### Sync
- **What**: Apply Git state to cluster
- **DOES** change cluster state
- **Use**: Deploy changes
- **Triggers**: Manual or automated

### Hard Refresh
- **What**: Flush cache and re-fetch everything
- **Use**: When you suspect cache issues
- **Command**: `argocd app get <name> --hard-refresh`

## Reconciliation Triggers

Reconciliation happens when:
1. **Timer expires** (every 3 minutes)
2. **Webhook received** (Git push)
3. **Manual refresh** (user clicks refresh)
4. **Application created/updated**
5. **Cluster events** (resource changes)

## Performance Considerations

### Large Repositories
- Reconciliation can be slow
- Consider: Sparse checkout, shallow clone
- Or: Split into multiple smaller repos

### Many Applications
- Each app reconciles independently
- Tune reconciliation timeout
- Consider: Application sets for bulk operations

### Resource Consumption
- Repository Server: CPU for manifest generation
- Application Controller: Memory for state tracking
- Redis: Cache storage

## Troubleshooting Reconciliation

### Slow Reconciliation
```bash
# Check controller logs
kubectl logs -n argocd deployment/argocd-application-controller

# Check repo server logs
kubectl logs -n argocd deployment/argocd-repo-server

# Common causes:
# - Large repo
# - Complex Helm charts
# - Network issues
# - Resource constraints
```

### Reconciliation Failures
```bash
# Check application events
kubectl describe application <name> -n argocd

# Check sync operation
argocd app get <name>

# Force reconciliation
argocd app get <name> --refresh --hard-refresh
```
