# Reconciliation Testing

## Test 1: Default Polling (3 minute interval)

### Setup
1. Application already created and synced
2. Will make change in Git
3. Observe how long until ArgoCD detects it

### Steps
```bash
# Check current state
argocd app get guestbook-ui

# Record time
echo "Change made at: Sun Jan 18 09:43:07 IST 2026"

# Make change in Git
cd ../argocd-demo-apps
# Add annotation to trigger change
cat <<EOL >> guestbook-app/manifests/deployment.yaml
  # Changed at Sun Jan 18 09:43:07 IST 2026
EOL
git add .
git commit -m "Test reconciliation timing"
git push

# Watch for detection
cd ../argo-mastery
while true; do
  STATUS=$(argocd app get guestbook-ui --refresh -o json | jq -r '.status.sync.status')
  echo "$(date): Status = $STATUS"
  if [ "$STATUS" = "OutOfSync" ]; then
    echo "Detected! Time taken: ~3 minutes"
    break
  fi
  sleep 10
done
```

### Result
- Time from push to detection: ~0-3 minutes
- Depends on where we are in 3-minute cycle
- Average: 1.5 minutes

## Test 2: Manual Refresh (Instant)

### Steps
```bash
# Make another change
cd ../argocd-demo-apps
echo "# Manual refresh test Sun Jan 18 09:43:07 IST 2026" >> guestbook-app/manifests/deployment.yaml
git add .
git commit -m "Test manual refresh"
git push

# Immediately refresh
cd ../argo-mastery
argocd app get guestbook-ui --refresh

# Check status immediately
argocd app get guestbook-ui
# Should show OutOfSync instantly!
```

### Result
- Detection: Instant
- No waiting for 3-minute cycle

## Test 3: Hard Refresh (Clear Cache)

### When to use
- Suspect caching issues
- Want to force complete re-fetch
- Troubleshooting

### Steps
```bash
argocd app get guestbook-ui --hard-refresh
```

## Test 4: Reconciliation Timeout

### Increase timeout for large repos
```bash
kubectl edit configmap argocd-cm -n argocd

# Add/modify:
data:
  timeout.reconciliation: 300s  # 5 minutes instead of 3
```

### Restart application controller
```bash
kubectl rollout restart deployment argocd-application-controller -n argocd
```
