# ApplicationSet Hands-On Demo

## Exercise 1: List Generator - Multi-Environment

```bash
# Create ApplicationSet for deploying to multiple environments
cat <<EOFAPP > argocd/labs/app-definitions/guestbook-appset.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook-environments
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - env: dev
        replicas: "1"
        namespace: guestbook-dev
      - env: staging
        replicas: "2"
        namespace: guestbook-staging
      - env: prod
        replicas: "3"
        namespace: guestbook-prod
  
  template:
    metadata:
      name: 'guestbook-{{env}}'
      labels:
        environment: '{{env}}'
        managed-by: applicationset
    spec:
      project: default
      source:
        repoURL: https://github.com/YOUR_USERNAME/argocd-demo-apps.git
        targetRevision: HEAD
        path: guestbook-app/manifests
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
EOFAPP

# Apply
kubectl apply -f argocd/labs/app-definitions/guestbook-appset.yaml

# Watch applications being created
watch kubectl get applications -n argocd

# Should see:
# - guestbook-dev
# - guestbook-staging
# - guestbook-prod

# Verify apps
argocd app list

# Check each environment
kubectl get all -n guestbook-dev
kubectl get all -n guestbook-staging
kubectl get all -n guestbook-prod
```

## Exercise 2: Cluster Generator

```bash
# If you have multiple clusters registered (from multi-cluster lab)
cat <<EOFAPP > /tmp/cluster-generator-appset.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook-all-clusters
  namespace: argocd
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          environment: development  # Only dev clusters
  
  template:
    metadata:
      name: 'guestbook-{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/YOUR_USERNAME/argocd-demo-apps.git
        targetRevision: HEAD
        path: guestbook-app/manifests
      destination:
        server: '{{server}}'
        namespace: guestbook
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
EOFAPP

# Note: This would create app on every registered cluster with that label
```

## Exercise 3: Git Directory Generator

```bash
# Create multi-tenant structure in Git
cd ../argocd-demo-apps
mkdir -p tenants/{team-a,team-b,team-c}

# Team A app
cat <<EOFAPP > tenants/team-a/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: team-a-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: team-a
  template:
    metadata:
      labels:
        app: team-a
    spec:
      containers:
      - name: app
        image: nginx:alpine
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
EOFAPP

# Similar for Team B and C
cp tenants/team-a/deployment.yaml tenants/team-b/deployment.yaml
sed -i '' 's/team-a/team-b/g' tenants/team-b/deployment.yaml

cp tenants/team-a/deployment.yaml tenants/team-c/deployment.yaml
sed -i '' 's/team-a/team-c/g' tenants/team-c/deployment.yaml

git add tenants/
git commit -m "Add multi-tenant structure"
git push

# Create ApplicationSet
cd ../../argo-mastery

cat <<EOFAPP > argocd/labs/app-definitions/tenants-appset.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: tenant-apps
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/YOUR_USERNAME/argocd-demo-apps.git
      revision: HEAD
      directories:
      - path: tenants/*
  
  template:
    metadata:
      name: '{{path.basename}}'
      labels:
        team: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/YOUR_USERNAME/argocd-demo-apps.git
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
EOFAPP

kubectl apply -f argocd/labs/app-definitions/tenants-appset.yaml

# Watch apps created
watch kubectl get applications -n argocd

# Should auto-create:
# - team-a
# - team-b
# - team-c

# Verify
kubectl get namespaces | grep team
kubectl get pods -n team-a
kubectl get pods -n team-b
kubectl get pods -n team-c
```

## Exercise 4: Test Auto-Generation

```bash
# Add Team D
cd ../argocd-demo-apps
mkdir -p tenants/team-d
cp tenants/team-a/deployment.yaml tenants/team-d/deployment.yaml
sed -i '' 's/team-a/team-d/g' tenants/team-d/deployment.yaml

git add tenants/team-d
git commit -m "Add team-d"
git push

# Wait ~3 minutes (reconciliation)
# Or force refresh
argocd appset get tenant-apps

# After sync, check
kubectl get application team-d -n argocd
kubectl get pods -n team-d

# Application automatically created!
```

## Exercise 5: Matrix Generator

```bash
# Deploy multiple apps to multiple environments
cat <<EOFAPP > argocd/labs/app-definitions/matrix-appset.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps-matrix
  namespace: argocd
spec:
  generators:
  - matrix:
      generators:
      - list:
          elements:
          - app: frontend
            port: "3000"
          - app: backend
            port: "8080"
      - list:
          elements:
          - env: dev
            replicas: "1"
          - env: prod
            replicas: "3"
  
  template:
    metadata:
      name: '{{app}}-{{env}}'
      labels:
        app: '{{app}}'
        environment: '{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/YOUR_USERNAME/argocd-demo-apps.git
        targetRevision: HEAD
        path: guestbook-app/manifests  # Using guestbook as placeholder
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{app}}-{{env}}'
      syncPolicy:
        automated:
          prune: true
        syncOptions:
          - CreateNamespace=true
EOFAPP

kubectl apply -f argocd/labs/app-definitions/matrix-appset.yaml

# Creates 4 applications:
# - frontend-dev
# - frontend-prod
# - backend-dev
# - backend-prod
```

## Cleanup

```bash
# Delete ApplicationSets
kubectl delete applicationset guestbook-environments -n argocd
kubectl delete applicationset tenant-apps -n argocd
kubectl delete applicationset apps-matrix -n argocd

# This will also delete generated applications (if syncPolicy allows)

# Or delete individually
argocd app delete guestbook-dev --cascade
argocd app delete team-a --cascade
# etc.
```

## Key Observations

1. **One ApplicationSet → Many Applications**
2. **Automatic generation** from patterns
3. **Git changes detected** and apps created
4. **Reduced maintenance** - modify template once
5. **Scalable** - works for 10 or 1000 apps

## Production Usage

In production, ApplicationSet enables:
- **Multi-cluster at scale** - 100+ clusters
- **Self-service for teams** - Add dir in Git = new app
- **Consistent deployment** - Same pattern everywhere
- **Disaster recovery** - Recreate all apps from Git
