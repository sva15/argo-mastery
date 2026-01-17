# Lab: Exploring ArgoCD Application and Project

## Objective
Understand application lifecycle and explore ArgoCD features

## Exercise 1: Application States

### Task 1: Observe OutOfSync State
1. Modify the application in Git but don't sync
```bash
cd ../argocd-demo-apps
# Change replicas in deployment
sed -i '' 's/replicas: 1/replicas: 3/' guestbook-app/manifests/deployment.yaml
git add .
git commit -m "Scale to 3 replicas"
git push
```

2. Refresh application in ArgoCD
```bash
argocd app get guestbook-ui --refresh
```

3. Observe OutOfSync status
   - Check UI: Shows diff between Git and cluster
   - CLI: `argocd app diff guestbook-ui`

### Task 2: Observe Synced State
1. Sync the application
```bash
argocd app sync guestbook-ui
```

2. Verify Synced status
   - UI shows green checkmark
   - `argocd app get guestbook-ui`

### Task 3: Create Manual Drift
1. Manually change something in cluster
```bash
kubectl scale deployment guestbook-ui -n dev --replicas=5
```

2. Wait 3 minutes (reconciliation period)
3. Observe: Still shows "Synced" but...
   - Live state: 5 replicas
   - Git state: 3 replicas
   - ArgoCD considers this "Synced" because it deployed what Git said
   - But cluster was manually changed after

4. With self-heal enabled, ArgoCD would revert this

## Exercise 2: Application Health

### Task 1: Healthy Application
1. Check current health: `argocd app get guestbook-ui`
2. Should show "Healthy"
3. In UI, drill down to see individual resource health

### Task 2: Progressing State
1. Trigger a rollout
```bash
kubectl set image deployment/guestbook-ui guestbook=gcr.io/heptio-images/ks-guestbook-demo:0.2 -n dev
```
2. Quickly check status: `argocd app get guestbook-ui`
3. Should show "Progressing" while rolling out

### Task 3: Degraded State (Simulate)
1. Break the application
```bash
cd ../argocd-demo-apps
# Use invalid image
sed -i '' 's|gcr.io/heptio-images/ks-guestbook-demo:0.1|invalid-image:latest|' guestbook-app/manifests/deployment.yaml
git add .
git commit -m "Use invalid image"
git push
```

2. Sync: `argocd app sync guestbook-ui`
3. Observe "Degraded" health
4. Check why: Pods failing to pull image

5. Fix it
```bash
git revert HEAD
git push
argocd app sync guestbook-ui
```

## Exercise 3: Application Details

### Explore in UI
1. Click on application
2. View **Resource Tree**
   - Shows all Kubernetes resources
   - Hierarchical view (Deployment → ReplicaSet → Pods)
3. Click on individual resources
   - View YAML
   - View events
   - View logs (for pods)

### Explore via CLI
```bash
# Full application details
argocd app get guestbook-ui

# Just diff
argocd app diff guestbook-ui

# Logs from application
argocd app logs guestbook-ui

# Specific resource logs
argocd app logs guestbook-ui --kind Pod --name guestbook-ui-xxx

# Events
kubectl get events -n dev --sort-by='.lastTimestamp'
```

## Exercise 4: Application History

### View History
```bash
# In UI: Application → History tab

# Via CLI
argocd app history guestbook-ui
```

### Shows:
- All sync operations
- Revision (Git commit)
- Who triggered sync
- Timestamp
- Status

### Rollback to Previous Version
```bash
# Get revision ID from history
argocd app history guestbook-ui

# Rollback to revision 1
argocd app rollback guestbook-ui 1
```

## Key Learnings
1. Application states: OutOfSync → Syncing → Synced
2. Health states: Progressing → Healthy/Degraded
3. Manual drift detection
4. Complete audit trail in history
5. Easy rollback capability

## Questions to Ponder
- What happens if Git server is down?
- How to handle emergency hotfixes?
- When to use manual vs auto sync?
