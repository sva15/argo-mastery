# ArgoCD Sync Hooks

## What are Sync Hooks?

Hooks are Kubernetes resources that execute at specific points during the sync process.

## Hook Types

### 1. PreSync
**When:** Before any resources are applied
**Use Cases:**
- Database migrations
- Backup current state
- Pre-deployment checks
- Schema updates

### 2. Sync
**When:** During normal sync with other resources
**Use Cases:**
- Normal application resources
- Default behavior (no annotation needed)

### 3. PostSync
**When:** After all Sync resources are healthy
**Use Cases:**
- Smoke tests
- Cache warming
- Notifications
- Health verification
- Data seeding

### 4. SyncFail
**When:** If sync operation fails
**Use Cases:**
- Rollback operations
- Send failure notifications
- Cleanup failed resources
- Alert on-call team

### 5. Skip
**When:** Never (resource ignored during sync)
**Use Cases:**
- Documentation
- Templates
- Resources managed elsewhere

## Hook Lifecycle

```
┌─────────────────┐
│   PreSync       │  ← Run first (DB migrations)
│   Jobs/Pods     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Sync          │  ← Main application resources
│   (Regular      │     (Deployments, Services, etc.)
│    Resources)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   PostSync      │  ← Run after everything healthy
│   Jobs/Pods     │     (Smoke tests, notifications)
└────────┬────────┘
         │
         ▼
    [Complete]

If ANY step fails:
         ↓
┌─────────────────┐
│   SyncFail      │  ← Run on failure
│   Jobs/Pods     │     (Cleanup, alerts)
└─────────────────┘
```

## Hook Resource Types

### Supported Resource Types
- **Job** (most common)
- **Pod**
- **CronJob** (for PostSync)
- Any Kubernetes resource with hook annotation

### Best Practice: Use Jobs
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: my-hook
  annotations:
    argocd.argoproj.io/hook: PreSync
```

**Why Jobs?**
- Automatic cleanup on completion
- Retry on failure
- Better status tracking
- Resource limits

## Hook Annotations

### Basic Hook
```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
```

### Hook Deletion Policy
```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

**Deletion Policies:**
- **HookSucceeded**: Delete after successful execution
- **HookFailed**: Delete after failed execution
- **BeforeHookCreation**: Delete existing hook before creating new one

**Multiple policies:**
```yaml
argocd.argoproj.io/hook-delete-policy: HookSucceeded,HookFailed
```

### Hook Weight (Execution Order)
```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-weight: "1"  # Lower runs first
```

**Example:**
- Weight -5: Database backup
- Weight 0: Schema migration
- Weight 5: Data migration

## Common Patterns

### Pattern 1: Database Migration
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
    argocd.argoproj.io/hook-weight: "0"
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: myapp:latest
        command: ["./migrate.sh"]
        env:
        - name: DB_HOST
          value: postgres
      restartPolicy: Never
  backoffLimit: 2
```

### Pattern 2: Smoke Test
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: smoke-test
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: test
        image: curlimages/curl
        command:
        - sh
        - -c
        - |
          curl -f http://myapp/health || exit 1
          echo "Health check passed!"
      restartPolicy: Never
  backoffLimit: 3
```

### Pattern 3: Backup Before Deploy
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-weight: "-1"  # Run before migrations
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: backup
        image: postgres:14
        command:
        - sh
        - -c
        - |
          pg_dump -h postgres -U user dbname > /backup/db-20260118-095419.sql
          echo "Backup complete"
      restartPolicy: Never
```

### Pattern 4: Notification on Success
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: notify-success
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: notify
        image: curlimages/curl
        command:
        - sh
        - -c
        - |
          curl -X POST https://slack-webhook-url             -H 'Content-Type: application/json'             -d '{"text":"Deployment successful!"}'
      restartPolicy: Never
```

### Pattern 5: Cleanup on Failure
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: cleanup-on-fail
  annotations:
    argocd.argoproj.io/hook: SyncFail
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: cleanup
        image: alpine
        command:
        - sh
        - -c
        - |
          echo "Sync failed! Performing cleanup..."
          # Cleanup operations here
          # Send alert
      restartPolicy: Never
```

## Hook Execution Flow

### Multiple Hooks Example
```yaml
# Hook 1: Weight -2
PreSync: backup (weight: -2)
↓
# Hook 2: Weight -1  
PreSync: db-schema-check (weight: -1)
↓
# Hook 3: Weight 0
PreSync: db-migrate (weight: 0)
↓
# Regular resources
Sync: Deployment, Service, ConfigMap
↓
# Wait for all to be healthy
↓
# Hook 4: Weight 0
PostSync: smoke-test (weight: 0)
↓
# Hook 5: Weight 1
PostSync: notify-success (weight: 1)
↓
Complete!
```

## Hook Failures

### What Happens When Hook Fails?

**PreSync Fails:**
- Sync operation stops immediately
- Regular resources NOT applied
- SyncFail hooks run
- Application status: Failed

**PostSync Fails:**
- Regular resources already applied
- Application deployed but marked as degraded
- SyncFail hooks run

**Best Practice:** Implement retries in hooks
```yaml
spec:
  backoffLimit: 3  # Retry up to 3 times
```

## Debugging Hooks

### Check Hook Status
```bash
# List all resources including hooks
argocd app get <app-name>

# View hook logs
argocd app logs <app-name> --kind Job --name <hook-job-name>

# Or directly
kubectl logs job/<hook-name> -n <namespace>

# Check hook events
kubectl describe job/<hook-name> -n <namespace>
```

### Common Hook Issues

**Issue 1: Hook never completes**
- Check restartPolicy: Should be Never or OnFailure
- Check backoffLimit
- Review hook logs for errors

**Issue 2: Hook deleted before inspection**
- Use appropriate deletion policy
- Don't use BeforeHookCreation during debugging

**Issue 3: Wrong execution order**
- Verify hook-weight values
- Lower weight = earlier execution

## Best Practices

### 1. Use Appropriate Resource Type
✅ **Job**: Most cases (recommended)
✅ **Pod**: Quick simple tasks
❌ **Deployment**: Don't use for hooks

### 2. Set Deletion Policies
```yaml
# Good: Clean up on success
argocd.argoproj.io/hook-delete-policy: HookSucceeded

# Better: Clean up on both success and failure
argocd.argoproj.io/hook-delete-policy: HookSucceeded,HookFailed

# Best during debugging: Keep for inspection
argocd.argoproj.io/hook-delete-policy: Never
```

### 3. Implement Idempotency
Hooks should be safe to run multiple times:
```sql
-- Bad: Will fail on second run
CREATE TABLE users;

-- Good: Idempotent
CREATE TABLE IF NOT EXISTS users;
```

### 4. Set Resource Limits
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

### 5. Implement Timeouts
```yaml
spec:
  activeDeadlineSeconds: 300  # 5 minute timeout
```

### 6. Use Weights for Ordering
```yaml
# Order: backup → validate → migrate
backup:     weight: -2
validate:   weight: -1
migrate:    weight: 0
```

### 7. Log Everything
```bash
#!/bin/bash
set -x  # Enable debug logging
echo "Starting hook execution..."
# Your commands here
echo "Hook completed successfully"
```

## Real-World Example: Complete Deployment Flow

```yaml
---
# PreSync: Backup database
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-db
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-weight: "-3"
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: backup
        image: postgres:14-alpine
        command: ["/scripts/backup.sh"]
      restartPolicy: Never
  backoffLimit: 1

---
# PreSync: Run database migrations
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-weight: "0"
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: myapp:v2.0.0
        command: ["npm", "run", "migrate"]
      restartPolicy: Never
  backoffLimit: 2

---
# Sync: Main application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  # ... deployment spec

---
# PostSync: Smoke test
apiVersion: batch/v1
kind: Job
metadata:
  name: smoke-test
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: test
        image: curlimages/curl
        command:
        - sh
        - -c
        - |
          for i in {1..5}; do
            if curl -f http://myapp/health; then
              echo "Health check passed"
              exit 0
            fi
            sleep 10
          done
          exit 1
      restartPolicy: Never
  backoffLimit: 3

---
# PostSync: Notify Slack
apiVersion: batch/v1
kind: Job
metadata:
  name: notify-slack
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-weight: "1"
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: notify
        image: curlimages/curl
        env:
        - name: SLACK_WEBHOOK
          valueFrom:
            secretKeyRef:
              name: slack-webhook
              key: url
        command:
        - sh
        - -c
        - |
          curl -X POST              -H 'Content-Type: application/json'             -d "{\"text\":\"✅ Deployment successful: myapp v2.0.0\"}"
      restartPolicy: Never

---
# SyncFail: Alert on failure
apiVersion: batch/v1
kind: Job
metadata:
  name: alert-on-fail
  annotations:
    argocd.argoproj.io/hook: SyncFail
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: alert
        image: curlimages/curl
        env:
        - name: SLACK_WEBHOOK
          valueFrom:
            secretKeyRef:
              name: slack-webhook
              key: url
        command:
        - sh
        - -c
        - |
          curl -X POST              -H 'Content-Type: application/json'             -d "{\"text\":\"🚨 Deployment FAILED: myapp v2.0.0\"}"
      restartPolicy: Never
```

## Hooks vs Sync Waves

**Hooks:**
- Jobs that run at specific lifecycle points
- Can be PreSync, Sync, PostSync, SyncFail
- Usually one-time operations

**Sync Waves:**
- Order of applying regular resources
- For orchestrating resource creation
- Covered in next section

**Often used together:**
```yaml
annotations:
  argocd.argoproj.io/hook: PreSync      # Lifecycle point
  argocd.argoproj.io/sync-wave: "0"     # Order within that phase
```
