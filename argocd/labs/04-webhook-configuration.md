# GitHub Webhook Configuration

## Why Webhooks?

Instead of polling every 3 minutes:
- Instant notification on Git push
- Reduces reconciliation delay to <1 second
- Reduces load on Git server
- More efficient

## Prerequisites

1. **ArgoCD must be publicly accessible**
   - Use ngrok for local testing
   - Or deploy ArgoCD to cloud cluster

2. **GitHub repository access**
   - Admin access to configure webhooks

## Step 1: Get ArgoCD Webhook URL

### URL Format
```
https://<argocd-server>/api/webhook
```

### For local setup with ngrok
```bash
# Install ngrok
brew install ngrok

# Expose ArgoCD
ngrok http 8080

# ngrok will provide URL like: https://abc123.ngrok.io
# Webhook URL: https://abc123.ngrok.io/api/webhook
```

## Step 2: Create Webhook Secret

```bash
# Generate random secret
WEBHOOK_SECRET=$(openssl rand -hex 32)
echo "Webhook Secret: $WEBHOOK_SECRET"

# Store in ArgoCD config
kubectl patch secret argocd-secret -n argocd -p="{\"data\":{\"webhook.github.secret\":\"$(echo -n $WEBHOOK_SECRET | base64)\"}}"
```

## Step 3: Configure in GitHub

### Via GitHub UI
1. Go to repository settings
2. Navigate to: Settings → Webhooks → Add webhook

3. Configure:
   - **Payload URL**: `https://your-argocd/api/webhook`
   - **Content type**: `application/json`
   - **Secret**: [paste webhook secret]
   - **SSL verification**: Enable (use valid cert)
   - **Events**: 
     - Just the push event (default)
     - Or select: Pushes, Pull Requests
   - **Active**: ✓ Checked

4. Click "Add webhook"

### Via GitHub API
```bash
curl -X POST \
  -H "Authorization: token <GITHUB_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "web",
    "active": true,
    "events": ["push"],
    "config": {
      "url": "https://your-argocd/api/webhook",
      "content_type": "json",
      "secret": "'""'"
    }
  }' \
  https://api.github.com/repos/YOUR_USERNAME/argocd-demo-apps/hooks
```

## Step 4: Test Webhook

### Make a change
```bash
cd ../argocd-demo-apps
echo "# Webhook test Sun Jan 18 09:43:48 IST 2026" >> guestbook-app/manifests/deployment.yaml
git add .
git commit -m "Test webhook"
git push
```

### Check webhook delivery
1. GitHub: Settings → Webhooks → Click on webhook
2. See "Recent Deliveries"
3. Should show successful delivery (green checkmark)

### Check ArgoCD received it
```bash
# Should detect change almost instantly (< 1 second)
argocd app get guestbook-ui --refresh
```

## Webhook Payload

GitHub sends POST request:
```json
{
  "ref": "refs/heads/main",
  "repository": {
    "url": "https://github.com/user/repo"
  },
  "pusher": {
    "name": "username"
  },
  "commits": [...]
}
```

ArgoCD processes:
1. Verifies signature (using webhook secret)
2. Extracts repo URL and branch
3. Finds applications watching that repo
4. Triggers immediate reconciliation

## Troubleshooting

### Webhook not received
```bash
# Check API server logs
kubectl logs -n argocd deployment/argocd-server | grep webhook

# Common issues:
# - ArgoCD not publicly accessible
# - Wrong webhook URL
# - Invalid signature (wrong secret)
# - SSL certificate issues
```

### Webhook received but not triggering sync
```bash
# Check application controller logs
kubectl logs -n argocd deployment/argocd-application-controller

# Verify application watches correct repo/branch
argocd app get <app-name> -o json | jq '.spec.source'
```

## Production Considerations

1. **Use valid SSL certificates**
   - Don't disable SSL verification
   - Use Let's Encrypt or proper certs

2. **Secure webhook secret**
   - Use strong random secret
   - Store in sealed secrets or vault

3. **Rate limiting**
   - GitHub has webhook rate limits
   - Configure reasonable thresholds

4. **Multiple webhooks**
   - Can configure different webhooks
   - For different events or repos

5. **Webhook logging**
   - Enable for troubleshooting
   - Monitor failed deliveries

## Alternative: GitLab Webhooks

Similar process for GitLab:
```
GitLab: Settings → Webhooks
URL: https://argocd/api/webhook
Secret Token: <webhook-secret>
Trigger: Push events
```

## Benefits Achieved

With webhooks:
- ✅ Detection time: <1 second (vs 0-3 minutes)
- ✅ Reduced Git server load
- ✅ Faster feedback loop
- ✅ Better developer experience
