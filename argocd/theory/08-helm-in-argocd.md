# Helm Charts in ArgoCD

## Why Use Helm with ArgoCD?

### Helm Benefits
- **Templating**: Parameterized manifests
- **Packaging**: Bundle related resources
- **Reusability**: Same chart, different values
- **Versioning**: Chart versions

### ArgoCD + Helm Benefits
- **GitOps for Helm**: Helm charts managed via Git
- **No Tiller**: ArgoCD generates manifests, applies directly
- **Version Control**: Values files in Git
- **Consistency**: Same deployment process as plain YAML

## How ArgoCD Uses Helm

### Traditional Helm
```
helm install myapp ./chart --values values.yaml
↓
Tiller/Helm → Kubernetes
```

### ArgoCD with Helm
```
ArgoCD → Template chart with values → Generate manifests → Apply to K8s
```

**Key Difference:** ArgoCD acts as Helm client, templates manifests, then applies them like any other resource.

## Helm Source Types in ArgoCD

### 1. Git Repository with Helm Chart
```yaml
source:
  repoURL: https://github.com/user/repo.git
  targetRevision: HEAD
  path: charts/myapp
  helm:
    valueFiles:
      - values.yaml
      - values-prod.yaml
```

### 2. Helm Repository
```yaml
source:
  repoURL: https://charts.bitnami.com/bitnami
  chart: nginx
  targetRevision: 13.2.0
  helm:
    parameters:
      - name: replicaCount
        value: "3"
```

### 3. OCI Registry (Helm 3.8+)
```yaml
source:
  repoURL: oci://registry.example.com/charts
  chart: myapp
  targetRevision: 1.0.0
```

## Helm Parameters

### Method 1: values.yaml Files
```yaml
spec:
  source:
    helm:
      valueFiles:
        - values.yaml          # Base values
        - values-production.yaml  # Environment overrides
```

### Method 2: Inline Parameters
```yaml
spec:
  source:
    helm:
      parameters:
        - name: replicaCount
          value: "5"
        - name: image.tag
          value: "v2.0.0"
        - name: service.type
          value: "LoadBalancer"
```

### Method 3: Values Block
```yaml
spec:
  source:
    helm:
      values: |
        replicaCount: 5
        image:
          repository: myapp
          tag: v2.0.0
        service:
          type: LoadBalancer
          port: 80
```

### Priority Order
1. **values block** (highest priority)
2. **parameters**
3. **valueFiles** in order listed
4. **chart's default values.yaml** (lowest priority)

## Complete Example

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: wordpress-prod
  namespace: argocd
spec:
  project: default
  
  # Helm chart from Git
  source:
    repoURL: https://github.com/bitnami/charts.git
    targetRevision: main
    path: bitnami/wordpress
    
    helm:
      # Values files (in order)
      valueFiles:
        - values.yaml
        - values-production.yaml
      
      # Override specific parameters
      parameters:
        - name: wordpressUsername
          value: admin
        - name: wordpressPassword
          value: production-password
        - name: replicaCount
          value: "3"
      
      # Or use values block
      values: |
        mariadb:
          enabled: true
          auth:
            rootPassword: root-password
            database: wordpress
        
        persistence:
          enabled: true
          size: 10Gi
        
        ingress:
          enabled: true
          hostname: wordpress.example.com
  
  destination:
    server: https://kubernetes.default.svc
    namespace: wordpress
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Helm Release Name

### Default Behavior
ArgoCD uses Application name as Helm release name:
```yaml
metadata:
  name: myapp  # This becomes Helm release name
```

### Custom Release Name
```yaml
spec:
  source:
    helm:
      releaseName: custom-release-name
```

### Why It Matters
- Resource names include release name
- `{{ .Release.Name }}` in templates
- Helm list shows this name

## Helm Hooks vs ArgoCD Sync Hooks

### Helm Hooks (Native)
```yaml
metadata:
  annotations:
    helm.sh/hook: pre-install,pre-upgrade
    helm.sh/hook-weight: "0"
    helm.sh/hook-delete-policy: before-hook-creation
```

### ArgoCD Sync Hooks
```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-weight: "0"
```

### Which to Use?
- **Helm hooks**: If chart might be used outside ArgoCD
- **ArgoCD hooks**: For ArgoCD-specific workflows
- **Both**: Can coexist (ArgoCD respects Helm hooks)

## Parameter Override Patterns

### Pattern 1: Environment-Specific Values Files

```
charts/myapp/
├── Chart.yaml
├── values.yaml           # Base values
├── values-dev.yaml       # Dev overrides
├── values-staging.yaml   # Staging overrides
└── values-prod.yaml      # Production overrides
```

```yaml
# Dev Application
spec:
  source:
    helm:
      valueFiles:
        - values.yaml
        - values-dev.yaml

# Prod Application
spec:
  source:
    helm:
      valueFiles:
        - values.yaml
        - values-prod.yaml
```

### Pattern 2: Single Source, Multiple Apps

```yaml
# Base chart in Git
source:
  repoURL: https://github.com/company/charts.git
  path: myapp

# Dev app - different parameters
helm:
  parameters:
    - name: environment
      value: dev
    - name: replicas
      value: "1"

# Prod app - different parameters
helm:
  parameters:
    - name: environment
      value: prod
    - name: replicas
      value: "5"
```

### Pattern 3: Public Chart, Custom Values

```yaml
# Use public Helm chart
source:
  repoURL: https://charts.bitnami.com/bitnami
  chart: postgresql
  targetRevision: 12.1.0

# Your custom values in Git
source:
  # ... chart info above
  helm:
    valueFiles:
      - https://raw.githubusercontent.com/company/config/main/postgres-values.yaml
```

## Working with Helm Repositories

### Add Helm Repository
```bash
# Via CLI
argocd repo add https://charts.bitnami.com/bitnami --type helm --name bitnami

# List repos
argocd repo list
```

### Declarative Repository
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bitnami-helm-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: helm
  name: bitnami
  url: https://charts.bitnami.com/bitnami
```

### Private Helm Repository
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-helm-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: helm
  name: private-charts
  url: https://charts.company.com
  username: user
  password: password
```

## Helm Chart Versions

### Specific Version
```yaml
source:
  chart: nginx
  targetRevision: 13.2.0  # Specific version
```

### Version Ranges
```yaml
source:
  chart: nginx
  targetRevision: "~13.2.0"  # Tilde range (~13.2.0 = >=13.2.0 <13.3.0)
  targetRevision: "^13.2.0"  # Caret range (^13.2.0 = >=13.2.0 <14.0.0)
```

### Latest
```yaml
source:
  chart: nginx
  targetRevision: "*"  # Latest version (not recommended for prod)
```

## Helm Template Rendering

### View Generated Manifests
```bash
# See what Helm will generate
argocd app manifests myapp

# Or using Helm directly
helm template myapp ./chart --values values.yaml
```

### Common Template Issues

**Issue 1: Indentation**
```yaml
# Wrong
spec:
  template:
{{ toYaml .Values.podSpec }}

# Right
spec:
  template:
{{ toYaml .Values.podSpec | indent 4 }}
```

**Issue 2: Missing Values**
```yaml
# Fails if .Values.optional not set
image: {{ .Values.optional.image }}

# Safe with default
image: {{ .Values.optional.image | default "nginx:latest" }}
```

## Debugging Helm in ArgoCD

### Check Template Rendering
```bash
# Get application details
argocd app get myapp

# View manifests
argocd app manifests myapp

# Check for errors
kubectl get application myapp -n argocd -o yaml
```

### Common Errors

**Error: "chart not found"**
- Helm repo not added to ArgoCD
- Wrong chart name
- Chart version doesn't exist

**Error: "failed to load values"**
- Values file doesn't exist in repo
- Typo in values file name
- Wrong path

**Error: "template: myapp/templates/deployment.yaml: executing..."**
- Template syntax error
- Missing required value
- Type mismatch

## Best Practices

### 1. Version Everything
```yaml
# Good: Specific versions
source:
  targetRevision: v1.2.3  # Git tag
  chart: nginx
  targetRevision: 13.2.0  # Chart version

# Bad: Moving targets
source:
  targetRevision: main  # Branch can change
  targetRevision: "*"   # Gets latest
```

### 2. Keep Values in Git
```yaml
# Good: Values in version control
helm:
  valueFiles:
    - values.yaml
    - values-prod.yaml

# Bad: Inline values (hard to track changes)
helm:
  values: |
    # 100 lines of config
```

### 3. Use Helm Dependencies
```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: 12.1.0
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

### 4. Lint Charts
```bash
# Before committing
helm lint ./chart

# Template with values
helm template myapp ./chart --values values.yaml | kubectl apply --dry-run=client -f -
```

### 5. Document Values
```yaml
# values.yaml
## @param replicaCount Number of replicas
## @param image.repository Image repository
## @param image.tag Image tag
```

## ArgoCD-Specific Helm Features

### Skip CRD Installation
```yaml
spec:
  source:
    helm:
      skipCrds: true  # Don't install CRDs from chart
```

### Pass Release Name
```yaml
spec:
  source:
    helm:
      releaseName: my-custom-release
```

### File Parameters
```yaml
spec:
  source:
    helm:
      fileParameters:
        - name: config
          path: /path/to/config.json
```

## Migration from Helm CLI to ArgoCD

### Step 1: Export Current Release
```bash
# Get current values
helm get values myapp -n namespace > current-values.yaml

# Get chart info
helm list -n namespace
```

### Step 2: Create Git Repo
```bash
# Store chart and values in Git
myrepo/
├── charts/
│   └── myapp/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
└── values/
    ├── values-dev.yaml
    └── values-prod.yaml
```

### Step 3: Create ArgoCD Application
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  source:
    repoURL: https://github.com/company/helm-charts.git
    path: charts/myapp
    helm:
      valueFiles:
        - ../../values/values-prod.yaml
  destination:
    namespace: namespace
```

### Step 4: Uninstall Helm Release
```bash
# After ArgoCD takes over
helm uninstall myapp -n namespace
```

## Real-World Example: WordPress Deployment

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: wordpress-prod
  namespace: argocd
spec:
  project: production
  
  source:
    # Using Bitnami WordPress chart
    repoURL: https://charts.bitnami.com/bitnami
    chart: wordpress
    targetRevision: 15.2.0
    
    helm:
      # Environment-specific values
      parameters:
        - name: wordpressUsername
          value: admin
        
        - name: wordpressEmail
          value: admin@company.com
        
        - name: replicaCount
          value: "3"
        
        - name: ingress.enabled
          value: "true"
        
        - name: ingress.hostname
          value: blog.company.com
        
        - name: persistence.size
          value: 20Gi
        
        - name: mariadb.primary.persistence.size
          value: 10Gi
      
      # Additional complex config
      values: |
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        
        mariadb:
          auth:
            database: wordpress_prod
          
          primary:
            resources:
              requests:
                cpu: 250m
                memory: 256Mi
        
        metrics:
          enabled: true
  
  destination:
    server: https://kubernetes.default.svc
    namespace: wordpress
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: false  # Manual approval for prod
    syncOptions:
      - CreateNamespace=true
  
  # Health check for WordPress
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # HPA manages this
