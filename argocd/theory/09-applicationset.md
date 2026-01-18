# ArgoCD ApplicationSet

## What is ApplicationSet?

**ApplicationSet** is a Kubernetes controller that automatically generates ArgoCD Applications based on templates and generators.

## The Problem It Solves

### Without ApplicationSet (Manual)

Need to deploy same app to 10 clusters:
```yaml
# Manually create 10 applications
app-cluster-1.yaml
app-cluster-2.yaml
...
app-cluster-10.yaml
```

**Problems:**
- Lots of duplication
- Error-prone
- Hard to maintain
- Doesn't scale

### With ApplicationSet (Automated)

```yaml
# One ApplicationSet generates all 10
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
spec:
  generators:
  - list:
      elements:
      - cluster: cluster-1
      - cluster: cluster-2
      # ... up to cluster-10
  
  template:
    # Application template
    # {{cluster}} gets replaced for each element
```

**Benefits:**
- ✅ Single source of truth
- ✅ Automatic application creation
- ✅ Easy to add new clusters
- ✅ Consistent configuration

## ApplicationSet Architecture

```
┌────────────────────────────┐
│   ApplicationSet           │
│   ┌──────────────────────┐ │
│   │  Generators          │ │  → Discovers targets
│   └──────────────────────┘ │
│   ┌──────────────────────┐ │
│   │  Template            │ │  → Defines app structure
│   └──────────────────────┘ │
└──────────┬─────────────────┘
           │
           ▼
    ┌─────────────────┐
    │  Applications   │
    │  Auto-generated │
    │  - app-1        │
    │  - app-2        │
    │  - app-3        │
    └─────────────────┘
```

## Generator Types

### 1. List Generator

Explicitly list parameters:
```yaml
generators:
- list:
    elements:
    - cluster: dev
      url: https://dev-cluster:6443
      environment: development
    - cluster: prod
      url: https://prod-cluster:6443
      environment: production
```

**Use Case:** Small, known set of targets

### 2. Cluster Generator

Auto-discover registered ArgoCD clusters:
```yaml
generators:
- clusters:
    selector:
      matchLabels:
        environment: production
```

**Use Case:** Deploy to all (or filtered) registered clusters

### 3. Git Generator - Files

Read parameters from files in Git:
```yaml
generators:
- git:
    repoURL: https://github.com/company/config.git
    revision: HEAD
    files:
    - path: "clusters/*.yaml"
```

**Use Case:** Cluster configs stored in Git

### 4. Git Generator - Directories

Generate app per directory:
```yaml
generators:
- git:
    repoURL: https://github.com/company/apps.git
    revision: HEAD
    directories:
    - path: "apps/*"
```

**Use Case:** Multi-tenant, one app per team

### 5. Matrix Generator

Combine multiple generators:
```yaml
generators:
- matrix:
    generators:
    - list:  # Apps
        elements:
        - app: app1
        - app: app2
    - list:  # Environments
        elements:
        - env: dev
        - env: prod
# Generates: app1-dev, app1-prod, app2-dev, app2-prod
```

**Use Case:** Deploy multiple apps to multiple environments

### 6. Merge Generator

Merge parameters from multiple generators:
```yaml
generators:
- merge:
    generators:
    - clusters: {}  # Get cluster info
    - list:         # Add custom params
        elements:
        - cluster: dev
          replicas: "1"
        - cluster: prod
          replicas: "5"
```

### 7. SCM Provider Generator

Discover repos from GitHub/GitLab:
```yaml
generators:
- scmProvider:
    github:
      organization: myorg
      allBranches: true
```

**Use Case:** Auto-discover all repos in org

## Template Structure

### Basic Template
```yaml
template:
  metadata:
    name: '{{cluster}}-guestbook'
  spec:
    project: default
    source:
      repoURL: https://github.com/argoproj/argocd-example-apps.git
      targetRevision: HEAD
      path: guestbook
    destination:
      server: '{{url}}'
      namespace: guestbook
```

### Template with Helm
```yaml
template:
  metadata:
    name: '{{app}}-{{environment}}'
  spec:
    source:
      helm:
        parameters:
        - name: environment
          value: '{{environment}}'
        - name: replicas
          value: '{{replicas}}'
```

## Complete Examples

### Example 1: Deploy to All Clusters

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook-all-clusters
  namespace: argocd
spec:
  generators:
  - clusters: {}  # All registered clusters
  
  template:
    metadata:
      name: '{{name}}-guestbook'
      labels:
        cluster: '{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps.git
        targetRevision: HEAD
        path: guestbook
      destination:
        server: '{{server}}'
        namespace: guestbook
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

### Example 2: Multi-Environment Deployment

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-environments
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - env: dev
        cluster: dev-cluster
        replicas: "1"
        valuesFile: values-dev.yaml
      - env: staging
        cluster: staging-cluster
        replicas: "2"
        valuesFile: values-staging.yaml
      - env: prod
        cluster: prod-cluster
        replicas: "5"
        valuesFile: values-prod.yaml
  
  template:
    metadata:
      name: 'myapp-{{env}}'
      labels:
        environment: '{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/company/myapp.git
        targetRevision: HEAD
        path: helm/myapp
        helm:
          valueFiles:
          - '{{valuesFile}}'
          parameters:
          - name: replicaCount
            value: '{{replicas}}'
      destination:
        name: '{{cluster}}'
        namespace: myapp
      syncPolicy:
        automated:
          prune: true
```

### Example 3: Git Directory Generator (Multi-Tenant)

```yaml
# Repo structure:
# apps/
#   team-a/
#     deployment.yaml
#   team-b/
#     deployment.yaml
#   team-c/
#     deployment.yaml

apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: team-apps
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/company/team-apps.git
      revision: HEAD
      directories:
      - path: apps/*
  
  template:
    metadata:
      name: '{{path.basename}}'  # team-a, team-b, etc.
    spec:
      project: default
      source:
        repoURL: https://github.com/company/team-apps.git
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
        syncOptions:
          - CreateNamespace=true
```

### Example 4: Matrix Generator (Apps × Environments)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps-matrix
  namespace: argocd
spec:
  generators:
  - matrix:
      generators:
      # Dimension 1: Applications
      - list:
          elements:
          - app: frontend
            port: "3000"
          - app: backend
            port: "8080"
          - app: api-gateway
            port: "80"
      
      # Dimension 2: Environments
      - list:
          elements:
          - env: dev
            cluster: dev-cluster
            replicas: "1"
          - env: prod
            cluster: prod-cluster
            replicas: "3"
  
  template:
    metadata:
      name: '{{app}}-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/company/apps.git
        targetRevision: HEAD
        path: '{{app}}'
        helm:
          parameters:
          - name: replicaCount
            value: '{{replicas}}'
          - name: service.port
            value: '{{port}}'
      destination:
        name: '{{cluster}}'
        namespace: '{{app}}-{{env}}'
```

## Template Variables

### Standard Variables

From cluster generator:
- `{{name}}` - Cluster name
- `{{server}}` - Cluster API URL
- `{{metadata.labels.xyz}}` - Cluster labels
- `{{metadata.annotations.xyz}}` - Cluster annotations

From git generator:
- `{{path}}` - Full path
- `{{path.basename}}` - Last part of path
- `{{path[0]}}`, `{{path[1]}}` - Path segments

### Custom Variables

From list generator:
- Any field you define:
```yaml
elements:
- cluster: dev
  myCustomField: value  # Access as {{myCustomField}}
```

## ApplicationSet Lifecycle

### Application Creation
1. ApplicationSet controller watches for ApplicationSet resources
2. Generators discover targets
3. Template rendered for each target
4. Applications created in ArgoCD

### Application Updates
1. Generator detects change (new cluster, new directory, etc.)
2. Template re-rendered
3. Applications updated or created
4. Removed targets → Applications deleted (if policy allows)

### Application Deletion

Control via `applicationsyncPolicy`:
```yaml
spec:
  syncPolicy:
    applicationsyncPolicy: create-only  # Don't delete apps
    # or
    applicationsyncPolicy: create-update  # Update but don't delete
    # or
    applicationsyncPolicy: create-delete  # Full sync (default)
```

## Use Cases

### 1. Multi-Cluster Deployment
Deploy same app to 100 clusters with one ApplicationSet

### 2. Multi-Tenant Platform
One directory per team → one app per team

### 3. Monorepo Management
Auto-discover microservices in monorepo

### 4. Progressive Rollout
Deploy to dev → staging → prod automatically

### 5. Disaster Recovery
Recreate all apps in new cluster from ApplicationSet

### 6. Configuration Management
Cluster configs in Git → Applications generated

## Best Practices

### 1. Use Meaningful Names
```yaml
template:
  metadata:
    name: '{{cluster}}-{{app}}-{{env}}'  # Clear and unique
```

### 2. Add Labels
```yaml
template:
  metadata:
    labels:
      managed-by: applicationset
      cluster: '{{cluster}}'
      environment: '{{env}}'
```

### 3. Control Deletion
```yaml
spec:
  syncPolicy:
    applicationsyncPolicy: create-update  # Safer than create-delete
```

### 4. Use Selectors
```yaml
generators:
- clusters:
    selector:
      matchLabels:
        environment: production  # Only prod clusters
```

### 5. Template Validation
Test template rendering before applying:
```bash
kubectl apply --dry-run=client -f applicationset.yaml
```

## Troubleshooting

### Issue: Applications Not Created
```bash
# Check ApplicationSet status
kubectl get applicationset -n argocd

# Describe for events
kubectl describe applicationset <name> -n argocd

# Check controller logs
kubectl logs -n argocd deployment/argocd-applicationset-controller
```

### Issue: Wrong Parameters
```bash
# Check generated applications
kubectl get applications -n argocd -l <label-from-template>

# View specific application
kubectl get application <name> -n argocd -o yaml
```

### Issue: Template Syntax Error
- Variables must be in quotes: `'{{var}}'`
- Check for typos in variable names
- Ensure generator provides all variables used

## Migration from Manual to ApplicationSet

### Step 1: Identify Pattern
Look at existing applications - what's common?

### Step 2: Extract Variables
Identify what changes between apps:
- Cluster
- Environment
- App name
- Configuration values

### Step 3: Choose Generator
- Few, known targets → List generator
- All clusters → Cluster generator
- Git-based config → Git generator

### Step 4: Create Template
Replace variables with placeholders

### Step 5: Test
Apply ApplicationSet, verify apps created correctly

### Step 6: Delete Manual Apps
Once confident, delete old manual applications

## Limitations

- ❌ **Cannot reference secrets** - Variables must be from generators
- ❌ **Limited logic** - No conditional templates (use generators)
- ❌ **Sync can be slow** - Many apps = longer sync times
- ❌ **Debugging harder** - Indirection makes troubleshooting complex

## When NOT to Use ApplicationSet

- **Single application** - Overhead not worth it
- **Highly custom apps** - Each very different from others
- **Complex logic needed** - ApplicationSet templates are simple
- **Learning ArgoCD** - Start with basic Applications first

## Key Takeaways

1. **Automation for Applications** - Generate apps from patterns
2. **Reduce duplication** - Single template, many applications
3. **Dynamic discovery** - Auto-detect clusters, repos, directories
4. **Scalable** - Manage hundreds of applications easily
5. **GitOps-friendly** - ApplicationSet itself in Git
6. **Powerful generators** - List, Cluster, Git, Matrix, etc.
