# Lab: ArgoCD App Reconciliation - Comprehensive Exercises

## Exercise 1: Understanding OutOfSync Conditions

### Task 1.1: Create Drift in Different Ways
```bash
# Method 1: Change in Git (normal flow)
cd ../argocd-demo-apps
echo "# Git change" >> guestbook-app/manifests/deployment.yaml
git add . && git commit -m "Git change" && git push

# Method 2: Manual kubectl change (drift)
kubectl patch deployment guestbook-ui -n dev -p '{"spec":{"replicas":7}}'

# Method 3: Delete resource manually
kubectl delete service guestbook-ui -n dev
```

### Task 1.2: Observe Behavior
- Check ArgoCD UI: What's the status?
- With selfHeal: What happens?
- Without selfHeal: What happens?

## Exercise 2: Sync with Different Options

### Task 2.1: Dry Run Sync
```bash
# See what would change WITHOUT actually changing it
argocd app sync guestbook-ui --dry-run

# Useful for:
# - Previewing changes
# - Reviewing before production sync
# - Understanding differences
```

### Task 2.2: Selective Sync
```bash
# Sync only specific resource
argocd app sync guestbook-ui --resource apps:Deployment:guestbook-ui

# Sync multiple specific resources
argocd app sync guestbook-ui \
  --resource apps:Deployment:guestbook-ui \
  --resource v1:Service:guestbook-ui
```

### Task 2.3: Force Sync
```bash
# Force re-creation of resources
argocd app sync guestbook-ui --force

# When to use:
# - Resources in bad state
# - Need to ensure clean state
# - Troubleshooting
```

### Task 2.4: Async Sync
```bash
# Start sync but don't wait
argocd app sync guestbook-ui --async

# Useful for:
# - CI/CD pipelines
# - Multiple parallel syncs
# - Long-running syncs
```

## Exercise 3: Health Monitoring

### Task 3.1: Break Application Health
```bash
# Use invalid image
cd ../argocd-demo-apps
sed -i '' 's|gcr.io/heptio-images/ks-guestbook-demo:0.1|invalid:latest|' \
  guestbook-app/manifests/deployment.yaml
git add . && git commit -m "Break health" && git push

# Watch health degradation
argocd app get guestbook-ui --watch
# Should show: Degraded

# Check why
kubectl describe pod -n dev -l app=guestbook
# Image pull error
```

### Task 3.2: Fix and Observe Recovery
```bash
git revert HEAD
git push
argocd app sync guestbook-ui
# Watch health recover
```

## Exercise 4: Prune Testing

### Task 4.1: Add Resource
```bash
# Add configmap
cat <<EOL > ../argocd-demo-apps/guestbook-app/manifests/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-config
data:
  key: value
EOL

git add . && git commit -m "Add configmap" && git push
# Sync and verify it's created
```

### Task 4.2: Remove and Test Prune
```bash
# Remove from Git
rm ../argocd-demo-apps/guestbook-app/manifests/configmap.yaml
git add . && git commit -m "Remove configmap" && git push

# With prune enabled: ConfigMap deleted automatically
# Without prune: ConfigMap remains (orphaned)

# Check orphaned resources
argocd app get guestbook-ui -o json | jq '.status.resources[] | select(.orphaned == true)'
```

## Exercise 5: Self-Heal Testing

### Task 5.1: Manual Changes with Self-Heal Disabled
```bash
# Create app without self-heal
cat <<EOL | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: no-selfheal
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/argocd-demo-apps.git
    targetRevision: HEAD
    path: guestbook-app/manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: test-selfheal
  syncPolicy:
    automated:
      prune: false
      selfHeal: false  # Disabled
    syncOptions:
      - CreateNamespace=true
EOL

# Sync it
argocd app sync no-selfheal

# Make manual change
kubectl scale deployment guestbook-ui -n test-selfheal --replicas=10

# Observe: Change persists!
# ArgoCD shows OutOfSync but doesn't fix it
```

### Task 5.2: Enable Self-Heal and Compare
```bash
# Update to enable self-heal
kubectl patch application no-selfheal -n argocd --type=merge \
  -p '{"spec":{"syncPolicy":{"automated":{"selfHeal":true}}}}'

# Make same manual change
kubectl scale deployment guestbook-ui -n test-selfheal --replicas=10

# Observe: Within 3 minutes, scales back to original
watch kubectl get deployment guestbook-ui -n test-selfheal
```

## Exercise 6: Reconciliation Timing

### Task 6.1: Measure Default Reconciliation
```bash
# Make change and time detection
echo "Change time: 1768710158" > /tmp/change-time.txt
cd ../argocd-demo-apps
echo "# Change Sun Jan 18 09:52:38 IST 2026" >> guestbook-app/manifests/deployment.yaml
git add . && git commit -m "Timing test" && git push

# Poll for detection
while true; do
  if argocd app get guestbook-ui --refresh | grep -q "OutOfSync"; then
    DETECT_TIME=1768710158
    CHANGE_TIME=
    ELAPSED=0
    echo "Detected in  seconds"
    break
  fi
  sleep 5
done
```

### Task 6.2: With Webhook (if configured)
- Should detect in <5 seconds
- Compare with polling method

## Exercise 7: Complex Sync Scenarios

### Task 7.1: Multiple Resources with Dependencies
```bash
# Create resources with dependencies
cat <<EOL > ../argocd-demo-apps/complex-app/manifests/all.yaml
# ConfigMap (should be created first)
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "postgres://db:5432"
---
# Deployment (depends on ConfigMap)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: nginx:alpine
        envFrom:
        - configMapRef:
            name: app-config
EOL

# Observe sync order
# ArgoCD should create ConfigMap before Deployment
```

## Exercise 8: Troubleshooting

### Common Issues and Solutions

#### Issue 1: Sync hangs
```bash
# Check sync operation
argocd app get <app> -o json | jq '.status.operationState'

# Check for hooks stuck
kubectl get pods -n <namespace> -l argocd.argoproj.io/instance=<app>

# Force terminate
argocd app terminate-op <app>
```

#### Issue 2: Resources not detected
```bash
# Hard refresh
argocd app get <app> --hard-refresh

# Check repo access
argocd repo list
kubectl logs -n argocd deployment/argocd-repo-server
```

#### Issue 3: Health check failing
```bash
# Check resource health
kubectl describe <resource-type> <name> -n <namespace>

# Check health check logs
kubectl logs -n argocd deployment/argocd-application-controller
```

## Exercise 9: Performance Testing

### Task 9.1: Sync Many Apps
```bash
# Create 10 similar apps
for i in {1..10}; do
  argocd app create test-app- \
    --repo https://github.com/YOUR_USERNAME/argocd-demo-apps.git \
    --path guestbook-app/manifests \
    --dest-server https://kubernetes.default.svc \
    --dest-namespace test- \
    --sync-option CreateNamespace=true
done

# Sync all at once
for i in {1..10}; do
  argocd app sync test-app- --async &
done
wait

# Observe resource usage
kubectl top pods -n argocd
```

## Lab Completion Checklist
- [ ] Tested all sync options
- [ ] Understood prune vs no-prune
- [ ] Tested self-heal behavior
- [ ] Measured reconciliation timing
- [ ] Created and fixed degraded health
- [ ] Practiced troubleshooting
- [ ] Observed performance characteristics

## Key Takeaways
1. Self-heal enforces GitOps but can surprise users
2. Prune is powerful but dangerous
3. Reconciliation default is 3 minutes
4. Health checks are customizable
5. Dry-run is your friend
6. Webhooks dramatically improve speed
