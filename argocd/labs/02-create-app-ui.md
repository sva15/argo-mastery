# Creating Application via UI

## Steps

1. **Open ArgoCD UI**
   - URL: https://localhost:30080
   - Login with admin credentials

2. **Click "+ NEW APP"**

3. **Fill Application Details**
   - **Application Name:** guestbook-ui
   - **Project:** default
   - **Sync Policy:** Manual (for now)

4. **Source Section**
   - **Repository URL:** https://github.com/sva15/argocd-demo-apps.git
   - **Revision:** HEAD
   - **Path:** guestbook-app/manifests

5. **Destination Section**
   - **Cluster URL:** https://kubernetes.default.svc
   - **Namespace:** dev

6. **Click CREATE**

## What Happens
- Application appears in "OutOfSync" state
- ArgoCD has compared Git with cluster
- Resources don't exist yet in cluster
- Waiting for sync

## Screenshots
[screenshot of application in OutOfSync state]

![OutOfSync](images\out-of-sync.png)

## Notes
- Application created but not deployed yet
- Manual sync required (we didn't enable auto-sync)
- Can see diff between desired and actual state
