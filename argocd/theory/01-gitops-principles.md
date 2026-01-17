# GitOps Principles - My Understanding

## The Four Principles

### 1. Declarative

* System state described declaratively (YAML, not scripts)
* Desired state vs procedural commands
* Example: Kubernetes manifests

### 2. Versioned and Immutable

* Git as single source of truth
* Complete history of all changes
* Easy rollback to any previous state

### 3. Pulled Automatically

* Software agents pull desired state from Git
* No push access needed to production
* More secure than traditional CI/CD

### 4. Continuously Reconciled

* Agents constantly compare actual vs desired state
* Auto-correct drift (self-healing)
* Ensures consistency

---

## Traditional CI/CD vs GitOps

**Traditional:**
CI builds → CI pushes to K8s → Hope it worked

**GitOps:**
CI builds → Update Git → ArgoCD pulls → K8s updated

* Git = source of truth
* Audit trail automatic
* Rollback = git revert

---

## Why This Matters

GitOps helps maintain consistency between the **desired state** and the **actual state** of the system.
Git acts as a **single source of truth**, providing a complete audit trail and easy rollback.

Even if someone makes a manual change directly in the cluster, continuous reconciliation detects the drift and restores the state defined in Git. If something goes wrong after deployment, ArgoCD can either automatically reconcile back to the last known good state or we can manually revert the Git commit to restore stability.

---

## What Is a Network Partition?

A **network partition** occurs when a network is split into two or more isolated segments, causing parts of a distributed system to lose communication with each other even though the components are still running.

Common causes include:

* Network device failures (routers, switches)
* Connectivity issues between data centers or regions
* Misconfigurations or transient network outages

During a partition, each segment continues to operate independently but cannot coordinate state or exchange updates until connectivity is restored.

---

## What Happens During a Network Partition (GitOps / ArgoCD)

ArgoCD follows a **pull-based reconciliation model**, where it periodically fetches the desired state from Git and compares it with the live cluster state.

During a network partition:

* If ArgoCD cannot reach the Git repository, it cannot pull new changes.
* If ArgoCD cannot reach the Kubernetes API, it cannot apply or reconcile changes.
* The cluster continues running with the **last successfully applied state**.
* No automatic rollback or corruption occurs due to the partition itself.

Once connectivity is restored, ArgoCD resumes reconciliation and brings the cluster back in sync with the desired state defined in Git.

---

## Research Questions

### How does ArgoCD handle secrets?

ArgoCD does **not provide a built-in encrypted secret store** and is intentionally unopinionated about secret management. Storing raw Kubernetes Secrets directly in Git is discouraged.

Recommended approaches include:

* Managing secrets **outside Git** and injecting them at runtime
* Using tools such as Sealed Secrets, External Secrets Operator, or secret managers like Vault or cloud-native secret services
* Letting ArgoCD deploy references to secrets rather than secret values themselves

Official documentation:
[https://argo-cd.readthedocs.io/en/stable/operator-manual/secret-management/](https://argo-cd.readthedocs.io/en/stable/operator-manual/secret-management/)

---

### What happens during network partition?

A network partition isolates parts of the system so they cannot communicate. In GitOps, this mainly affects **synchronization**, not correctness. ArgoCD simply pauses reconciliation until connectivity is restored, after which it continues enforcing the desired state from Git.

---

### How to handle database migrations?

Common approaches in GitOps workflows include:

* Using **PreSync or PostSync hooks** to run migration jobs
* Managing schema changes through a **dedicated database migration operator**
* Running migrations in CI before updating Git manifests

These approaches help ensure migrations are controlled, repeatable, and do not unintentionally re-run on every reconciliation.
