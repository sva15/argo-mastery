# Manual Sync Process

## Sync Command
```bash
argocd app sync guestbook-ui --prune
```

## Sync Options
- `--prune`: Delete resources not in Git
- `--dry-run`: Preview without applying
- `--force`: Force sync even if no changes
- `--async`: Don't wait for completion
- `--timeout`: Timeout for sync operation

## Wait for Sync
```bash
argocd app wait guestbook-ui
```

## What Happened During Sync

### Phase 1: PreSync
- ArgoCD checked for PreSync hooks
- None found in our case

### Phase 2: Sync
- Created namespace 'dev' (CreateNamespace option)
- Applied deployment.yaml
  - Created Deployment guestbook-ui
  - Created ReplicaSet
  - Created Pod(s)
- Applied service.yaml
  - Created Service guestbook-ui

### Phase 3: PostSync
- Checked for PostSync hooks
- None found

### Phase 4: Health Assessment
- Waited for Deployment to become healthy
- Checked replica count matches spec
- Verified all pods are Running
- Service endpoint created

## Sync Result
- Status: Synced
- Health: Healthy
- Resources: 2 (Deployment, Service)
- Sync Time: ~30 seconds

## Verification
```bash
# ArgoCD view
argocd app get guestbook-ui

# Kubernetes view
kubectl get all -n dev

# Application access
curl http://localhost:30000
```

## Screenshots
[screenshot of synced application in ArgoCD UI]

![Synced](images\sync.png)
