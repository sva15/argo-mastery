# ArgoCD Multi-Cluster Deployment

## What is Multi-Cluster?

Single ArgoCD instance managing applications across multiple Kubernetes clusters.

## Architecture

```
┌─────────────────────────────────────┐
│  ArgoCD Control Plane Cluster      │
│                                     │
│  ┌──────────────┐                  │
│  │  ArgoCD      │                  │
│  │  Server      │                  │
│  └──────┬───────┘                  │
│         │                           │
└─────────┼───────────────────────────┘
          │
          ├──────────────┬──────────────┬──────────────┐
          │              │              │              │
          ▼              ▼              ▼              ▼
    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
    │  Dev    │    │ Staging │    │  Prod   │    │ Prod    │
    │ Cluster │    │ Cluster │    │ US-East │    │ US-West │
    └─────────┘    └─────────┘    └─────────┘    └─────────┘
```

## Use Cases

### 1. Environment Separation
- Dev cluster
- Staging cluster  
- Production cluster
- Each isolated, different configurations

### 2. Geographic Distribution
- US-East cluster
- US-West cluster
- EU cluster
- Asia cluster
- Same app, multiple regions

### 3. Multi-Tenancy
- Team A cluster
- Team B cluster
- Team C cluster
- Isolation and resource control

### 4. Hybrid/Multi-Cloud
- On-premise cluster
- AWS cluster
- GCP cluster
- Azure cluster

## Cluster Registration

### Method 1: Using CLI (Most Common)

```bash
# Login to ArgoCD
argocd login <argocd-server>

# Add external cluster
argocd cluster add <context-name>

# List registered clusters
argocd cluster list

# Example:
argocd cluster add kind-dev-cluster
argocd cluster add kind-staging-cluster
argocd cluster add kind-prod-cluster
```

### Method 2: Using Declarative Config

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dev-cluster-secret
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: dev-cluster
  server: https://dev-cluster-api:6443
  config: |
    {
      "bearerToken": "<token>",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "<base64-ca-cert>"
      }
    }
```

### What Happens During Registration

1. **Service Account Created** in target cluster
   - Name: argocd-manager
   - Namespace: kube-system

2. **ClusterRole Created** with necessary permissions
   - Get, list, watch resources
   - Create, update, patch, delete

3. **ClusterRoleBinding** connects SA to role

4. **Secret Created** in ArgoCD namespace
   - Contains cluster API endpoint
   - Contains service account token
   - Contains CA certificate

## Application Deployment to Specific Cluster

### Single Application to Specific Cluster

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/user/repo.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://dev-cluster-api:6443  # Specific cluster
    namespace: default
```

### Same App to Multiple Clusters

```yaml
---
# Dev deployment
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-dev
spec:
  destination:
    server: https://dev-cluster-api:6443
    namespace: myapp
  source:
    path: overlays/dev  # Dev-specific config
---
# Staging deployment
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-staging
spec:
  destination:
    server: https://staging-cluster-api:6443
    namespace: myapp
  source:
    path: overlays/staging
---
# Production deployment
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
spec:
  destination:
    server: https://prod-cluster-api:6443
    namespace: myapp
  source:
    path: overlays/prod
```

## Cluster Identification

### By Server URL
```yaml
destination:
  server: https://kubernetes.default.svc  # In-cluster
  server: https://dev-cluster:6443        # External cluster
```

### By Name
```yaml
destination:
  name: dev-cluster  # Cluster name (if set)
  namespace: default
```

## Multi-Cluster Patterns

### Pattern 1: Hub and Spoke
- Single ArgoCD in management cluster
- Deploys to multiple workload clusters
- Centralized control and visibility

### Pattern 2: Per-Environment ArgoCD
- ArgoCD in dev cluster → manages dev
- ArgoCD in prod cluster → manages prod
- Complete isolation

### Pattern 3: Federated
- Multiple ArgoCD instances
- Each manages subset of clusters
- Coordination through external tools

## Security Considerations

### Cluster Permissions

ArgoCD needs permissions in target clusters:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argocd-manager-role
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ''
  - apps
  - batch
  resources:
  - '*'
  verbs:
  - '*'
```

**Production:** Should use least-privilege principle!

### Network Connectivity

ArgoCD must reach target cluster APIs:
- Network paths must exist
- Firewalls configured
- VPNs if needed
- Private endpoints secured

### Credential Management

Cluster credentials stored in Kubernetes secrets:
- Use strong RBAC on ArgoCD namespace
- Consider external secret management (Vault)
- Rotate credentials regularly
- Audit access

## Monitoring Multi-Cluster

### Cluster Health
```bash
# Check cluster connectivity
argocd cluster list

# Example output:
# SERVER                          NAME        VERSION  STATUS   MESSAGE
# https://kubernetes.default.svc  in-cluster  1.28     Successful
# https://dev-cluster:6443        dev         1.28     Successful
# https://prod-cluster:6443       prod        1.28     Connection timeout
```

### Application Distribution
```bash
# See apps per cluster
argocd app list -o wide

# Filter by cluster
argocd app list --cluster https://dev-cluster:6443
```

## Troubleshooting Multi-Cluster

### Issue 1: Cannot Connect to Cluster

```bash
# Check cluster status
argocd cluster list

# Test connectivity
kubectl --context <cluster-context> get nodes

# Check ArgoCD logs
kubectl logs -n argocd deployment/argocd-application-controller

# Common causes:
# - Network unreachable
# - Certificate issues
# - Token expired
# - API endpoint changed
```

### Issue 2: Permission Denied

```bash
# Check service account permissions
kubectl --context <cluster-context> auth can-i create deployments --as=system:serviceaccount:kube-system:argocd-manager

# Review ClusterRole
kubectl --context <cluster-context> get clusterrole argocd-manager-role -o yaml
```

### Issue 3: Wrong Cluster

```bash
# Application deployed to wrong cluster
# Check destination in app spec
argocd app get <app-name> -o yaml | grep -A5 destination

# List all clusters
argocd cluster list

# Update application destination
argocd app set <app-name> --dest-server https://correct-cluster:6443
```

## Best Practices

### 1. Naming Conventions
```bash
# Clear cluster names
dev-us-east
staging-us-west
prod-eu-central
prod-asia-southeast
```

### 2. Project Boundaries
```yaml
# Restrict which clusters a project can use
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a
spec:
  destinations:
  - namespace: 'team-a-*'
    server: https://dev-cluster:6443
  - namespace: 'team-a-*'
    server: https://staging-cluster:6443
  # NOT allowed to deploy to prod
```

### 3. Cluster Labeling
```yaml
# Add labels to cluster secrets for easier filtering
metadata:
  labels:
    argocd.argoproj.io/secret-type: cluster
    environment: production
    region: us-east
    cloud-provider: aws
```

### 4. Disaster Recovery
- Document cluster registration process
- Keep cluster credentials in secure backup
- Test recovery procedures
- Have runbook for re-registration

### 5. RBAC Alignment
- Use same namespace structure across clusters
- Align RBAC policies
- Centralize identity management
- Audit regularly

## Real-World Example: Multi-Region Deployment

```yaml
---
# US-East Production
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ecommerce-prod-us-east
  labels:
    region: us-east
    environment: production
spec:
  project: production
  source:
    repoURL: https://github.com/company/ecommerce.git
    targetRevision: v2.1.0  # Specific version for prod
    path: k8s/overlays/prod
    helm:
      parameters:
      - name: region
        value: us-east
      - name: replicas
        value: "10"  # High traffic region
  destination:
    server: https://prod-us-east-cluster:6443
    namespace: ecommerce
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

---
# US-West Production  
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ecommerce-prod-us-west
  labels:
    region: us-west
    environment: production
spec:
  project: production
  source:
    repoURL: https://github.com/company/ecommerce.git
    targetRevision: v2.1.0
    path: k8s/overlays/prod
    helm:
      parameters:
      - name: region
        value: us-west
      - name: replicas
        value: "5"  # Medium traffic
  destination:
    server: https://prod-us-west-cluster:6443
    namespace: ecommerce
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

---
# EU Production
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ecommerce-prod-eu
  labels:
    region: eu-central
    environment: production
spec:
  project: production
  source:
    repoURL: https://github.com/company/ecommerce.git
    targetRevision: v2.1.0
    path: k8s/overlays/prod
    helm:
      parameters:
      - name: region
        value: eu-central
      - name: replicas
        value: "8"  # High traffic region
  destination:
    server: https://prod-eu-cluster:6443
    namespace: ecommerce
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Advantages of Multi-Cluster with ArgoCD

✅ **Single Control Plane** - Manage everything from one place
✅ **Consistent Deployments** - Same process for all clusters
✅ **Visibility** - See all apps across all clusters
✅ **GitOps Everywhere** - Same principles apply everywhere
✅ **Disaster Recovery** - Easy to recreate in new cluster
✅ **Compliance** - Centralized audit trail
✅ **Cost Optimization** - Better resource utilization

## Limitations

❌ **Single Point of Failure** - ArgoCD down = no deployments
❌ **Network Dependency** - Requires connectivity to all clusters
❌ **Scalability** - One ArgoCD managing 100+ clusters can struggle
❌ **Security Scope** - Compromise of ArgoCD = access to all clusters

## Migration Strategy

### Phase 1: Single Cluster (Current)
- ArgoCD managing local cluster only

### Phase 2: Add Dev/Staging
- Register development cluster
- Register staging cluster
- Test multi-cluster features

### Phase 3: Add Production
- Register production cluster
- Implement strict RBAC
- Enable audit logging

### Phase 4: Scale
- Add more regions
- Implement automation
- Monitor and optimize

## Key Takeaways

1. **One ArgoCD can manage many clusters**
2. **Cluster registration creates service account in target**
3. **Applications specify destination cluster**
4. **Use Projects to restrict cluster access**
5. **Monitor cluster connectivity**
6. **Plan for disaster recovery**
7. **Implement least-privilege RBAC**
