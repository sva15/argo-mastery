# ArgoCD CLI Setup

## Installation
```bash
$version = (Invoke-RestMethod https://api.github.com/repos/argoproj/argo-cd/releases/latest).tag_name

$url = "https://github.com/argoproj/argo-cd/releases/download/" + $version + "/argocd-windows-amd64.exe"
$output = "argocd.exe"

Invoke-WebRequest -Uri $url -OutFile $output

[Environment]::SetEnvironmentVariable("Path", "$env:Path;C:\Path\To\ArgoCD-CLI", "User")
```

## Login
```bash
argocd login localhost:30080 --username admin --password <password> --insecure
```

## Verify
```bash
argocd version
# Should show both client and server versions
```

## Useful CLI Commands
```bash
# List applications
argocd app list

# Get application details
argocd app get <app-name>

# Sync application
argocd app sync <app-name>

# Watch application
argocd app wait <app-name>

# Logs
argocd app logs <app-name>

# Diff
argocd app diff <app-name>

# Rollback
argocd app rollback <app-name>

# Delete
argocd app delete <app-name>
```

## Current Access
- UI: https://localhost:30080
- Username: admin
- Password: [see argocd-credentials.txt]
- CLI: Logged in and ready
