# Helm Charts in ArgoCD - Hands-On Guide

## Exercise 1: Deploy from Public Helm Repository

### Task 1.1: Deploy Nginx from Bitnami

```bash
# Add Bitnami Helm repository to ArgoCD
argocd repo add https://charts.bitnami.com/bitnami --type helm --name bitnami

# Verify
argocd repo list

# Create application using Helm chart
cat <<EOFAPP > argocd/labs/app-definitions/nginx-helm.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-helm-demo
  namespace: argocd
spec:
  project: default
  
  source:
    # Bitnami Helm repository
    repoURL: https://charts.bitnami.com/bitnami
    chart: nginx
    targetRevision: 15.0.0
    
    helm:
      # Override values
      parameters:
        - name: replicaCount
          value: "2"
        - name: service.type
          value: "NodePort"
        - name: service.nodePorts.http
          value: "30002"
      
      values: |
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 100m
            memory: 128Mi
  
  destination:
    server: https://kubernetes.default.svc
    namespace: nginx-demo
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOFAPP

# Apply
kubectl apply -f argocd/labs/app-definitions/nginx-helm.yaml

# Watch deployment
argocd app get nginx-helm-demo --watch

# View generated manifests
argocd app manifests nginx-helm-demo

# Access the application
curl http://localhost:30002
```

## Exercise 2: Create Custom Helm Chart

### Task 2.1: Build Your Own Chart

```bash
cd ../argocd-demo-apps
mkdir -p helm-charts
cd helm-charts

# Create new Helm chart
helm create myapp

# Customize the chart
cd myapp

# Edit Chart.yaml
cat <<EOFCHART > Chart.yaml
apiVersion: v2
name: myapp
description: Custom application Helm chart
type: application
version: 1.0.0
appVersion: "1.0"
EOFCHART

# Create custom values.yaml
cat <<EOFVAL > values.yaml
# Default values for myapp
replicaCount: 1

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "alpine"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}

# Custom application config
config:
  environment: development
  logLevel: info
  features:
    featureA: true
    featureB: false
EOFVAL

# Create environment-specific values
cat <<EOFDEV > values-dev.yaml
replicaCount: 1

image:
  tag: "latest"

config:
  environment: development
  logLevel: debug
  features:
    featureA: true
    featureB: true

resources:
  limits:
    cpu: 200m
    memory: 256Mi
EOFDEV

cat <<EOFPROD > values-prod.yaml
replicaCount: 3

image:
  tag: "1.0.0"

service:
  type: LoadBalancer

config:
  environment: production
  logLevel: warning
  features:
    featureA: true
    featureB: false

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi
EOFPROD

# Customize deployment template
cat <<EOFDEP > templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: 80
          protocol: TCP
        env:
        - name: ENVIRONMENT
          value: {{ .Values.config.environment | quote }}
        - name: LOG_LEVEL
          value: {{ .Values.config.logLevel | quote }}
        - name: FEATURE_A
          value: {{ .Values.config.features.featureA | quote }}
        - name: FEATURE_B
          value: {{ .Values.config.features.featureB | quote }}
        livenessProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
EOFDEP

# Add ConfigMap template
cat <<EOFCM > templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
data:
  app.conf: |
    environment: {{ .Values.config.environment }}
    log_level: {{ .Values.config.logLevel }}
    features:
      feature_a: {{ .Values.config.features.featureA }}
      feature_b: {{ .Values.config.features.featureB }}
EOFCM

# Test the chart
cd ..
helm lint myapp

# Test rendering
helm template myapp-dev ./myapp --values ./myapp/values-dev.yaml
helm template myapp-prod ./myapp --values ./myapp/values-prod.yaml

# Commit to Git
cd ..
git add helm-charts/
git commit -m "Add custom Helm chart with environment-specific values"
git push
```

### Task 2.2: Deploy Custom Chart via ArgoCD

```bash
cd ../../argo-mastery

# Create dev deployment
cat <<EOFAPP > argocd/labs/app-definitions/myapp-dev-helm.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-dev
  namespace: argocd
  labels:
    environment: development
spec:
  project: default
  
  source:
    repoURL: https://github.com/YOUR_USERNAME/argocd-demo-apps.git
    targetRevision: HEAD
    path: helm-charts/myapp
    
    helm:
      releaseName: myapp
      valueFiles:
        - values.yaml        # Base values
        - values-dev.yaml    # Dev overrides
  
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-dev
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOFAPP

# Create prod deployment (manual sync)
cat <<EOFAPP > argocd/labs/app-definitions/myapp-prod-helm.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
  namespace: argocd
  labels:
    environment: production
spec:
  project: default
  
  source:
    repoURL: https://github.com/YOUR_USERNAME/argocd-demo-apps.git
    targetRevision: HEAD
    path: helm-charts/myapp
    
    helm:
      releaseName: myapp
      valueFiles:
        - values.yaml        # Base values
        - values-prod.yaml   # Prod overrides
  
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-prod
  
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    # Manual sync for production
EOFAPP

# Apply both
kubectl apply -f argocd/labs/app-definitions/myapp-dev-helm.yaml
kubectl apply -f argocd/labs/app-definitions/myapp-prod-helm.yaml

# Watch dev auto-sync
argocd app get myapp-dev --watch

# Manually sync prod
argocd app sync myapp-prod
argocd app wait myapp-prod

# Verify deployments
kubectl get all -n myapp-dev
kubectl get all -n myapp-prod

# Check environment-specific config
kubectl get deployment -n myapp-dev -o yaml | grep -A5 env:
kubectl get deployment -n myapp-prod -o yaml | grep -A5 env:

# Dev should have: ENVIRONMENT=development, LOG_LEVEL=debug
# Prod should have: ENVIRONMENT=production, LOG_LEVEL=warning
```

## Exercise 3: Parameter Overrides

### Task 3.1: Override via Application

```bash
# Update dev app to change replicas without changing values file
argocd app set myapp-dev --helm-set replicaCount=3

# Verify
kubectl get deployment -n myapp-dev

# Reset to values file
argocd app unset myapp-dev --helm-set replicaCount
argocd app sync myapp-dev
```

### Task 3.2: Test Different Configurations

```bash
# Create staging app with inline values
cat <<EOFAPP > /tmp/myapp-staging-helm.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/argocd-demo-apps.git
    targetRevision: HEAD
    path: helm-charts/myapp
    helm:
      values: |
        replicaCount: 2
        
        image:
          tag: "1.0.0-rc1"
        
        config:
          environment: staging
          logLevel: info
          features:
            featureA: true
            featureB: true
        
        resources:
          limits:
            cpu: 300m
            memory: 384Mi
          requests:
            cpu: 150m
            memory: 192Mi
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-staging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOFAPP

kubectl apply -f /tmp/myapp-staging-helm.yaml
```

## Exercise 4: Helm Dependencies

### Task 4.1: Add PostgreSQL Dependency

```bash
cd ../argocd-demo-apps/helm-charts/myapp

# Add dependency to Chart.yaml
cat <<EOFCHART > Chart.yaml
apiVersion: v2
name: myapp
description: Custom application Helm chart
type: application
version: 1.1.0  # Bumped version
appVersion: "1.0"

# Dependencies
dependencies:
  - name: postgresql
    version: "12.1.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
EOFCHART

# Update values to configure PostgreSQL
cat <<EOFVAL >> values.yaml

# PostgreSQL dependency
postgresql:
  enabled: true
  auth:
    username: myapp
    password: myapp-password
    database: myapp
  primary:
    persistence:
      enabled: true
      size: 1Gi
EOFVAL

# Update dependencies
helm dependency update

# This downloads postgresql chart to charts/ directory

# Commit
git add .
git commit -m "Add PostgreSQL dependency to myapp chart"
git push

# ArgoCD will detect change and sync
# Now myapp includes PostgreSQL!
```

## Exercise 5: Debugging Helm Templates

### Task 5.1: View Rendered Manifests

```bash
# See what ArgoCD generated from Helm chart
argocd app manifests myapp-dev

# Compare environments
argocd app manifests myapp-dev > /tmp/dev-manifests.yaml
argocd app manifests myapp-prod > /tmp/prod-manifests.yaml
diff /tmp/dev-manifests.yaml /tmp/prod-manifests.yaml
```

### Task 5.2: Test Locally

```bash
cd ../argocd-demo-apps/helm-charts

# Template with values
helm template myapp-test ./myapp --values ./myapp/values-dev.yaml

# Validate against cluster
helm template myapp-test ./myapp --values ./myapp/values-dev.yaml | kubectl apply --dry-run=client -f -

# Debug mode
helm template myapp-test ./myapp --values ./myapp/values-dev.yaml --debug
```

## Key Learnings

✅ **ArgoCD acts as Helm client** - Templates charts, applies manifests
✅ **Multiple value sources** - Files, parameters, inline values
✅ **Environment-specific configs** - Different values per environment
✅ **GitOps for Helm** - Values and charts in version control
✅ **Dependencies work** - Helm chart dependencies supported
✅ **No Tiller needed** - ArgoCD handles everything

## Best Practices Demonstrated

1. **Version control values files** - All configs in Git
2. **Environment-specific values** - Clear separation
3. **Use dependencies** - Don't reinvent (PostgreSQL)
4. **Test before committing** - ==> Linting .
Error unable to check Chart.yaml file in chart: CreateFile Chart.yaml: The system cannot find the file specified. and 
5. **Semantic versioning** - Bump chart version on changes
6. **Document values** - Comments in values.yaml
7. **Resource limits** - Always set for production

## Cleanup

```bash
# Delete applications
argocd app delete nginx-helm-demo --cascade
argocd app delete myapp-dev --cascade
argocd app delete myapp-staging --cascade
argocd app delete myapp-prod --cascade
```
