# ArgoCD Architecture

## Core Components

### 1. API Server

* **Role**: REST/gRPC server for Web UI, CLI, CI/CD
* **Responsibilities**:

  * Application management
  * Authentication
  * RBAC enforcement
  * Git credential handling
* **Port**: 8080 (HTTP), 8083 (gRPC)

### 2. Repository Server

* **Role**: Internal service for Git repo operations
* **Responsibilities**:

  * Clones Git repositories
  * Generates K8s manifests (Helm, Kustomize, etc.)
  * Caches repository data
* **Why Separate**: Performance & scalability

### 3. Application Controller

* **Role**: Kubernetes controller that monitors running apps
* **Responsibilities**:

  * Continuously compares live state vs desired state (Git)
  * Detects OutOfSync status
  * Performs sync operations
  * Invokes hooks (PreSync, PostSync, etc.)
* **Reconciliation Loop**: Default 3 minutes

### 4. Redis

* **Role**: Cache layer
* **Stores**:

  * App state
  * Cached repo data
  * Temporary data

### 5. Dex (Optional)

* **Role**: Identity provider
* **Responsibilities**:

  * SSO integration
  * OAuth2/OIDC support

### 6. ApplicationSet Controller (Optional)

* **Role**: Automation for app generation
* **Use Case**: Multi-cluster, multi-tenant deployments

---

## Key Terminology

### Application

* Group of Kubernetes resources defined by manifest
* Links to Git repo + destination cluster

### Project

* Logical grouping of applications
* RBAC boundary
* Source repos restriction
* Destination clusters restriction

### Target State

* Desired state in Git repo

### Live State

* Actual state in Kubernetes cluster

### Sync Status

* **Synced**: Live state matches target state
* **OutOfSync**: Differences detected

### Health Status

* **Healthy**: All resources running correctly
* **Progressing**: Resources still starting
* **Degraded**: Some resources unhealthy
* **Missing**: Resources not found

### Refresh vs Sync

* **Refresh**: Compare Git vs K8s (read-only)
* **Sync**: Apply changes to make them match

---

## How It Works - The Flow

1. Developer pushes to Git
2. Repository Server detects change (webhook or poll)
3. Repository Server generates manifests
4. Application Controller compares with live state
5. If different → OutOfSync
6. If auto-sync enabled → Sync automatically
7. If manual sync → Wait for user action
8. Apply changes to Kubernetes
9. Monitor health status
10. Update application status in ArgoCD

---

## Questions Answered

### Why is the Repository Server separated from the Application Controller?

The Repository Server is separated to **offload Git operations and manifest generation** from the Application Controller.
Cloning repositories and rendering manifests (Helm, Kustomize) are CPU- and IO-intensive tasks. By isolating them:

* Application Controller stays lightweight and focused on reconciliation
* Git operations can be cached and reused
* The system scales better when managing many applications or large repositories

---

### How often does ArgoCD check for changes?

The Application Controller runs a **reconciliation loop every 3 minutes by default** (configurable).
Additionally:

* Git webhooks can trigger immediate refresh
* Manual refresh can be triggered by users
* Health checks happen continuously based on Kubernetes watch events

---

### What happens if Git is down?

If the Git repository becomes unavailable:

* Repository Server uses **cached repository data** from previous successful fetches
* No new desired state can be pulled
* ArgoCD continues comparing live state against the **last known target state**
* Syncs that require new Git data will fail until Git connectivity is restored

The cluster **does not roll back or break automatically** due to Git being temporarily unavailable.

---

### What happens if Kubernetes is down?

If the Kubernetes API server is unreachable:

* Application Controller cannot fetch live state
* Sync and health checks fail temporarily
* ArgoCD retries reconciliation using **exponential backoff**
* Desired state remains unchanged in Git

Once Kubernetes becomes reachable again, ArgoCD resumes reconciliation and brings the cluster back in sync.

---

## Architecture Diagram

```
┌─────────────┐
│   Git Repo  │ ◄─── Developer pushes
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│ Repository      │ ◄─── Watches Git
│ Server          │      Generates manifests
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Application     │ ◄─── Reconciliation loop
│ Controller      │      Compares states
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Kubernetes     │ ◄─── Applies changes
│  Cluster        │
└─────────────────┘
         ▲
         │
    ┌────┴─────┐
    │ API      │ ◄─── Users interact
    │ Server   │
    └──────────┘
```

---