# Creating Application via CLI

## Command
```bash
argocd app create guestbook-cli \
  --repo https://github.com/YOUR_USERNAME/argocd-demo-apps.git \
  --path guestbook-app/manifests \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev \
  --sync-option CreateNamespace=true
```

## Parameters Explained
- `guestbook-cli`: Application name
- `--repo`: Git repository URL
- `--path`: Path to manifests in repo
- `--dest-server`: Kubernetes cluster API server
- `--dest-namespace`: Target namespace
- `--sync-option`: Additional sync options

## Verification
```bash
argocd app list
argocd app get guestbook-cli
```

## Advantages of CLI
1. Quick for testing
2. Scriptable
3. Good for CI/CD pipelines
4. Easy to iterate during development

## Other Useful CLI Options
```bash
# With auto-sync
argocd app create myapp \
  ... \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# With revision
--revision v1.0.0

# With sync retry
--sync-retry-limit 5 \
--sync-retry-backoff-duration 5s \
--sync-retry-backoff-max-duration 3m

# With directory recursion
--directory-recurse
```
