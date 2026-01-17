# ArgoCD Projects

## What is a Project?

Projects provide:
1. **Logical grouping** of applications
2. **RBAC boundaries** - who can access what
3. **Source repo restrictions** - which repos can be used
4. **Destination restrictions** - where apps can be deployed
5. **Resource restrictions** - what can be deployed

## Projects Created

### 1. dev-project
- **Purpose**: Development environment
- **Allowed Repos**: Any GitHub repo from YOUR_USERNAME
- **Allowed Destinations**: dev, dev-* namespaces
- **Resource Policy**: Most resources allowed
- **Use Case**: Developer experimentation and testing

### 2. prod-project
- **Purpose**: Production environment
- **Allowed Repos**: Only argocd-demo-apps (specific repo)
- **Allowed Destinations**: Only production namespace
- **Resource Policy**: Strict whitelist
- **Use Case**: Controlled production deployments

### 3. team-frontend
- **Purpose**: Team-specific isolation
- **Allowed Repos**: Only frontend-* repos
- **Allowed Destinations**: Only frontend-* namespaces
- **RBAC Roles**: Developer, Admin
- **Use Case**: Team autonomy with boundaries

## Project Components

```yaml
spec:
  sourceRepos:           # Where manifests can come from
  destinations:          # Where apps can be deployed
  clusterResourceWhitelist:  # Cluster-scoped resources allowed
  namespaceResourceWhitelist: # Namespace-scoped resources allowed
  namespaceResourceBlacklist: # Explicitly denied resources
  orphanedResources:     # How to handle untracked resources
  roles:                 # RBAC roles for the project
```

## Why Use Projects?

### Multi-Tenancy
- Different teams, different projects
- Each team can only deploy to their namespaces
- Prevents accidental deployments to wrong environment

### Security
- Production project: Very restrictive
- Dev project: More permissive
- Secrets can be restricted to specific projects

### Organization
- Group related applications
- Easier to manage permissions
- Clear ownership

## Common Patterns

### Pattern 1: Environment-based Projects
- dev-project
- staging-project
- prod-project

### Pattern 2: Team-based Projects
- team-frontend
- team-backend
- team-platform

### Pattern 3: Application-type Projects
- web-apps
- batch-jobs
- databases

## Testing Project Restrictions

### Valid Application (works)
```yaml
project: dev-project
destination:
  namespace: dev  # Allowed by dev-project
```

### Invalid Application (fails)
```yaml
project: dev-project
destination:
  namespace: production  # NOT allowed by dev-project
```

## Project CLI Commands

```bash
# List projects
argocd proj list

# Get project details
argocd proj get <project-name>

# Create project
argocd proj create <project-name>

# Add source repo
argocd proj add-source <project-name> <repo-url>

# Add destination
argocd proj add-destination <project-name> <cluster> <namespace>

# Add role
argocd proj role create <project-name> <role-name>

# Delete project
argocd proj delete <project-name>
```

## Best Practices

1. **Use projects for RBAC** - Don't rely on namespace RBAC alone
2. **Principle of least privilege** - Prod should be most restrictive
3. **Document project purpose** - Use description field
4. **Review allowed sources** - Don't use wildcards in production
5. **Monitor orphaned resources** - Enable warnings
6. **Use roles for teams** - Define clear responsibilities

## Lab Exercises Complete
- ✅ Created 3 projects with different security levels
- ✅ Created application in dev-project
- ✅ Tested project restrictions
- ✅ Understood RBAC implications
