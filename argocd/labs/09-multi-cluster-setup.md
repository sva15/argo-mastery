# Multi-Cluster Setup Guide

## Simulating Multi-Cluster Locally

We'll create 3 Kind clusters:
1. **argocd-cluster** - Control plane (already have this)
2. **dev-cluster** - Development environment
3. **staging-cluster** - Staging environment

## Step 1: Create Additional Clusters

```bash
# Create dev cluster
cat <<EOFDEV > /tmp/dev-cluster-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: dev-cluster
nodes:
- role: control-plane
- role: worker
EOFDEV

kind create cluster --config /tmp/dev-cluster-config.yaml

# Create staging cluster
cat <<EOFSTG > /tmp/staging-cluster-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: staging-cluster
nodes:
- role: control-plane
- role: worker
EOFSTG

kind create cluster --config /tmp/staging-cluster-config.yaml

# Verify all clusters
kind get clusters
# Should show: argocd-learning, dev-cluster, staging-cluster

# Check contexts
kubectl config get-contexts
```

## Step 2: Register Clusters with ArgoCD

```bash
# Switch to ArgoCD cluster
kubectl config use-context kind-argocd-learning

# Verify ArgoCD is running
kubectl get pods -n argocd

# Register dev cluster
argocd cluster add kind-dev-cluster --name dev-cluster

# You'll see:
# - WARNING about cluster admin access
# - Creating ServiceAccount in kube-system
# - Creating ClusterRole
# - Creating ClusterRoleBinding
# - Cluster added successfully

# Register staging cluster
argocd cluster add kind-staging-cluster --name staging-cluster

# List all clusters
argocd cluster list

# Expected output:
# SERVER                          NAME            VERSION  STATUS      MESSAGE
# https://kubernetes.default.svc  in-cluster      1.28     Successful
# https://dev-cluster:6443        dev-cluster     1.28     Successful
# https://staging-cluster:6443    staging-cluster 1.28     Successful
```

## Step 3: Verify Registration

```bash
# Check service account in dev cluster
kubectl --context kind-dev-cluster get sa -n kube-system argocd-manager

# Check cluster role
kubectl --context kind-dev-cluster get clusterrole argocd-manager-role

# Check cluster role binding
kubectl --context kind-dev-cluster get clusterrolebinding argocd-manager-role-binding

# Check secret in ArgoCD namespace
kubectl get secret -n argocd | grep cluster
```

## Step 4: Deploy to Different Clusters

### Create Applications for Each Environment

```bash
# Dev environment application
cat <<EOFAPP > /tmp/guestbook-dev.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/argocd-demo-apps.git
    targetRevision: HEAD
    path: guestbook-app/manifests
  destination:
    name: dev-cluster  # Deploy to dev cluster
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOFAPP

kubectl apply -f /tmp/guestbook-dev.yaml

# Staging environment application
cat <<EOFAPP > /tmp/guestbook-staging.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/argocd-demo-apps.git
    targetRevision: HEAD
    path: guestbook-app/manifests
  destination:
    name: staging-cluster  # Deploy to staging cluster
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOFAPP

kubectl apply -f /tmp/guestbook-staging.yaml

# Production (in-cluster) application
cat <<EOFAPP > /tmp/guestbook-prod.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/argocd-demo-apps.git
    targetRevision: HEAD
    path: guestbook-app/manifests
  destination:
    server: https://kubernetes.default.svc  # In-cluster
    namespace: production
  syncPolicy:
    # Manual sync for production
    syncOptions:
      - CreateNamespace=true
EOFAPP

kubectl apply -f /tmp/guestbook-prod.yaml
```

## Step 5: Verify Deployments

```bash
# Check ArgoCD applications
argocd app list

# Should see all three applications

# Verify pods in dev cluster
kubectl --context kind-dev-cluster get pods -n default

# Verify pods in staging cluster
kubectl --context kind-staging-cluster get pods -n default

# Verify pods in prod (in-cluster)
kubectl --context kind-argocd-learning get pods -n production

# All should have guestbook pods running!
```

## Step 6: Test Multi-Cluster Sync

```bash
# Make a change to the app
cd ../argocd-demo-apps
sed -i '' 's/replicas: 1/replicas: 2/' guestbook-app/manifests/deployment.yaml
git add .
git commit -m "Scale to 2 replicas"
git push

# Watch all three environments sync
watch 'argocd app list'

# After ~3 minutes, all should sync
# Verify in each cluster
kubectl --context kind-dev-cluster get deployment guestbook-ui -n default
kubectl --context kind-staging-cluster get deployment guestbook-ui -n default
kubectl --context kind-argocd-learning get deployment guestbook-ui -n production
# All should show 2/2 replicas

# Revert change
git revert HEAD
git push
```

## Step 7: Environment-Specific Configurations

```bash
# In real scenarios, you'd have different configs per environment
# Using Kustomize overlays or Helm values

# Example structure:
# repo/
#   base/
#     deployment.yaml  (replicas: 1)
#     service.yaml
#   overlays/
#     dev/
#       kustomization.yaml  (replicas: 1)
#     staging/
#       kustomization.yaml  (replicas: 2)
#     prod/
#       kustomization.yaml  (replicas: 5, resources, etc.)

# Then applications point to different paths:
# dev-app: path: overlays/dev
# staging-app: path: overlays/staging
# prod-app: path: overlays/prod
```

## Step 8: Cluster Operations

### Remove Cluster
```bash
# Remove dev cluster from ArgoCD
argocd cluster rm dev-cluster

# This deletes:
# - Secret in ArgoCD namespace
# - Does NOT delete ServiceAccount in target cluster (you should clean up manually)

# Clean up manually
kubectl --context kind-dev-cluster delete sa argocd-manager -n kube-system
kubectl --context kind-dev-cluster delete clusterrole argocd-manager-role
kubectl --context kind-dev-cluster delete clusterrolebinding argocd-manager-role-binding
```

### Re-add Cluster
```bash
# Can re-add anytime
argocd cluster add kind-dev-cluster --name dev-cluster
```

## Step 9: Monitor Cluster Health

```bash
# Continuous monitoring
watch 'argocd cluster list'

# Check specific cluster
argocd cluster get dev-cluster

# View cluster info
argocd cluster get dev-cluster -o json | jq .info
```

## Cleanup (When Done)

```bash
# Delete applications
argocd app delete guestbook-dev --cascade
argocd app delete guestbook-staging --cascade
argocd app delete guestbook-prod --cascade

# Remove clusters from ArgoCD
argocd cluster rm dev-cluster
argocd cluster rm staging-cluster

# Delete Kind clusters
kind delete cluster --name dev-cluster
kind delete cluster --name staging-cluster

# Keep argocd-learning cluster for remaining sessions
```

## Key Learnings

✅ **One ArgoCD → Many Clusters**
✅ **Cluster registration creates ServiceAccount**
✅ **Applications specify destination cluster**
✅ **Same app can deploy to multiple clusters**
✅ **Each cluster can have different config**
✅ **Centralized visibility and control**

## Production Considerations

When implementing multi-cluster in production:

1. **Network Security**
   - Use private endpoints
   - Implement network policies
   - VPN/PrivateLink for cross-cloud

2. **RBAC**
   - Least-privilege service accounts
   - Use Projects to restrict access
   - Regular access reviews

3. **High Availability**
   - Multiple ArgoCD replicas
   - External Redis for HA
   - Disaster recovery plan

4. **Monitoring**
   - Alert on cluster connectivity issues
   - Monitor sync failures
   - Track deployment metrics per cluster

5. **Secrets Management**
   - Don't store cluster credentials in Git
   - Use external secret managers
   - Rotate credentials regularly
