# Cluster Setup Documentation

## Cluster Details
- Name: argocd-learning
- Nodes: 1 control-plane + 2 workers
- Created: Sat Jan 17 13:57:31 IST 2026

## Namespaces
- argocd: ArgoCD installation
- dev: Development environment
- staging: Staging environment
- production: Production environment

## Port Mappings
- 30080: ArgoCD Server (HTTP)
- 30443: ArgoCD Server (gRPC)
- 30000-30002: Application ports

## Verification Commands
```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get namespaces
```

## Cleanup Command
```bash
kind delete cluster --name argocd-learning
```
