# ArgoCD Installation

## Installation Method
Standard kubectl apply method for learning purposes.

## Components Installed

### Deployments
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS                         IMAGES                                             SELECTOR
argocd-applicationset-controller   1/1     1            1           4m31s   argocd-applicationset-controller   quay.io/argoproj/argocd:v3.2.5                     app.kubernetes.io/name=argocd-applicationset-controller
argocd-dex-server                  1/1     1            1           4m31s   dex                                ghcr.io/dexidp/dex:v2.43.0                         app.kubernetes.io/name=argocd-dex-server
argocd-notifications-controller    1/1     1            1           4m30s   argocd-notifications-controller    quay.io/argoproj/argocd:v3.2.5                     app.kubernetes.io/name=argocd-notifications-controller
argocd-redis                       1/1     1            1           4m30s   redis                              public.ecr.aws/docker/library/redis:8.2.2-alpine   app.kubernetes.io/name=argocd-redis
argocd-repo-server                 1/1     1            1           4m30s   argocd-repo-server                 quay.io/argoproj/argocd:v3.2.5                     app.kubernetes.io/name=argocd-repo-server
argocd-server                      1/1     1            1           4m30s   argocd-server                      quay.io/argoproj/argocd:v3.2.5                     app.kubernetes.io/name=argocd-server

### Services
NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   10.96.26.31     <none>        7000/TCP,8080/TCP            4m32s
argocd-dex-server                         ClusterIP   10.96.128.158   <none>        5556/TCP,5557/TCP,5558/TCP   4m31s
argocd-metrics                            ClusterIP   10.96.235.212   <none>        8082/TCP                     4m31s
argocd-notifications-controller-metrics   ClusterIP   10.96.22.25     <none>        9001/TCP                     4m31s
argocd-redis                              ClusterIP   10.96.177.105   <none>        6379/TCP                     4m31s
argocd-repo-server                        ClusterIP   10.96.157.83    <none>        8081/TCP,8084/TCP            4m31s
argocd-server                             ClusterIP   10.96.9.195     <none>        80/TCP,443/TCP               4m31s
argocd-server-metrics                     ClusterIP   10.96.151.149   <none>        8083/TCP                     4m31s

### Key Components
1. argocd-server: API Server & UI
2. argocd-repo-server: Repository operations
3. argocd-application-controller: Main reconciliation loop
4. argocd-redis: Caching layer
5. argocd-dex-server: SSO (optional)
6. argocd-notifications-controller: Notifications

## Installation Time
- Started: Sat Jan 17 14:03:21 IST 2026
- All pods ready in: ~2 minutes

## Initial Configuration
- Default namespace: argocd
- Default admin username: admin
- Password stored in secret: argocd-initial-admin-secret

## Production Considerations
For production, consider:
- Helm chart installation for easier management
- HA configuration (multiple replicas)
- External Redis for better performance
- Persistent volumes for cache
- Ingress configuration
- Custom SSL certificates
- SSO integration (Dex/OIDC)
