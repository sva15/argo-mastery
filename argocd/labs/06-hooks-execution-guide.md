# Sync Hooks Execution Guide

## Application: hooks-demo

### Hook Execution Order

```
PreSync Hooks (ordered by weight):
├── Weight -5: presync-check-db (Check database connectivity)
├── Weight -3: presync-backup (Backup current database)
└── Weight 0:  presync-migrate (Run migrations)
     ↓
Sync Phase:
├── StatefulSet: postgres
├── Service: postgres  
├── Deployment: backend-api
└── Service: backend-api
     ↓
PostSync Hooks (ordered by weight):
├── Weight 0: postsync-smoke-test (Run smoke tests)
├── Weight 1: postsync-seed-data (Seed initial data)
└── Weight 5: postsync-notify (Send success notification)

If ANY step fails:
└── SyncFail Hook: syncfail-alert (Send failure alert)
```

## Step-by-Step Observation

### Step 1: Sync the Application
```bash
# Start sync and watch
argocd app sync hooks-demo

# In another terminal, watch resources
watch kubectl get all,job -n hooks-demo
```

### Step 2: Observe PreSync Hooks

You should see jobs created in order:
```bash
# Check job status
kubectl get jobs -n hooks-demo

# Watch first hook (check-db)
kubectl logs job/presync-check-db -n hooks-demo -f

# Expected output:
# "Checking database connectivity..."
# "Database is ready!"

# Watch second hook (backup)
kubectl logs job/presync-backup -n hooks-demo -f

# Watch third hook (migrate)
kubectl logs job/presync-migrate -n hooks-demo -f
```

### Step 3: Observe Main Resource Creation

After all PreSync hooks succeed:
```bash
# Watch deployments being created
kubectl get deployments -n hooks-demo -w

# Watch pods coming up
kubectl get pods -n hooks-demo -w
```

### Step 4: Observe PostSync Hooks

After all main resources are healthy:
```bash
# Watch PostSync jobs
kubectl get jobs -n hooks-demo | grep postsync

# Check smoke test
kubectl logs job/postsync-smoke-test -n hooks-demo -f

# Check data seeding
kubectl logs job/postsync-seed-data -n hooks-demo -f

# Check notification
kubectl logs job/postsync-notify -n hooks-demo -f
```

### Step 5: Verify Database State

```bash
# Connect to database and verify migrations
kubectl exec -it postgres-0 -n hooks-demo -- psql -U appuser -d appdb

# In psql:
\dt                          # List tables
SELECT * FROM users;         # Check seeded data
SELECT * FROM posts;
\q                          # Quit
```

## Testing Hook Failure Scenarios

### Scenario 1: Migration Failure

```bash
# Edit migration to introduce error
cd ../argocd-demo-apps
# Add invalid SQL to migration job
# Commit and push
# Sync again
argocd app sync hooks-demo

# Should fail at PreSync and trigger SyncFail hook
kubectl logs job/syncfail-alert -n hooks-demo
```

### Scenario 2: PostSync Failure

```bash
# Make smoke test fail
# Edit to check wrong endpoint
# Sync should complete but app marked degraded
# SyncFail hook will run
```

## Hook Debugging

### View All Hook Logs
```bash
# List all jobs (including hooks)
kubectl get jobs -n hooks-demo

# Get logs from all hooks
for job in $(kubectl get jobs -n hooks-demo -o name); do
  echo "=== $job ==="
  kubectl logs $job -n hooks-demo
done
```

### Check Hook Events
```bash
kubectl get events -n hooks-demo --sort-by='.lastTimestamp'
```

### View in ArgoCD UI
1. Click on application
2. See resources with hook icon
3. Click on hook job to see logs
4. View execution timeline

## Cleanup and Retest

### Delete Application
```bash
argocd app delete hooks-demo --cascade
# or
kubectl delete application hooks-demo -n argocd
```

### Retest Hook Flow
```bash
# Recreate application
kubectl apply -f argocd/labs/app-definitions/hooks-demo-app.yaml

# Sync and watch entire flow again
argocd app sync hooks-demo --watch
```

## Key Observations

### Deletion Policies in Action

**BeforeHookCreation:**
- Old hook deleted before new one created
- Useful for always having fresh hook
- Example: presync-check-db

**HookSucceeded:**
- Deleted after successful completion
- Keeps failed hooks for debugging
- Example: Most PostSync hooks

### Hook Weights in Action

Notice the order:
1. Weight -5: Database check (must pass first)
2. Weight -3: Backup (before changes)
3. Weight 0: Migration (make changes)
4. Weight 0: Smoke test (verify)
5. Weight 1: Seed data (populate)
6. Weight 5: Notify (final step)

### Failure Behavior

**PreSync fails:**
- Everything stops
- No regular resources created
- Safe - nothing deployed

**PostSync fails:**
- App already deployed
- Marked as degraded
- SyncFail hook runs
- Manual intervention needed

## Real-World Patterns Demonstrated

✅ **Database readiness check** before operations
✅ **Backup before changes** for safety
✅ **Database migrations** before app deployment
✅ **Smoke tests** to verify deployment
✅ **Data seeding** for fresh environments
✅ **Notifications** on success/failure
✅ **Proper error handling** with SyncFail

## Exercise: Add Your Own Hook

Try adding:
1. PreSync hook to check external dependency
2. PostSync hook to warm cache
3. SyncFail hook to create incident ticket

```yaml
# Your hook here
apiVersion: batch/v1
kind: Job
metadata:
  name: my-custom-hook
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-weight: "3"
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  # ... your implementation
```
