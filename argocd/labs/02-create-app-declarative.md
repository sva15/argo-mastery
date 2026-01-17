# Creating Application Declaratively

## Application Manifest
See: `argocd/labs/app-definitions/guestbook-declarative.yaml`

## Apply Command
```bash
kubectl apply -f argocd/labs/app-definitions/guestbook-declarative.yaml
```

## Verification
```bash
# List all applications
argocd app list

# Get specific application details
argocd app get guestbook-declarative

# Check in Kubernetes
kubectl get application -n argocd guestbook-declarative
```

## Advantages of Declarative
1. Version controlled
2. Repeatable
3. Can manage ArgoCD apps with ArgoCD (App of Apps pattern)
4. CI/CD friendly
5. Easy to template with Helm/Kustomize

## Application Anatomy
```yaml
metadata:
  name: application-name         # Unique name
  namespace: argocd              # Must be argocd namespace
  finalizers:                    # Cleanup behavior
    - resources-finalizer...     # Deletes app resources when app deleted

spec:
  project: default               # ArgoCD project (RBAC boundary)
  
  source:                        # Where to get manifests
    repoURL: ...                 # Git repository
    targetRevision: HEAD         # Branch/tag/commit
    path: ...                    # Path in repo
  
  destination:                   # Where to deploy
    server: ...                  # K8s cluster API
    namespace: ...               # Target namespace
  
  syncPolicy:                    # How to sync
    automated: {}                # Auto-sync settings
    syncOptions: []              # Additional options
```
