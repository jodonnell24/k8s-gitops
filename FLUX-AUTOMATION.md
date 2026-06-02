# Flux Automation Guide

## How Flux Automatically Syncs (No Manual Commands Needed!)

### Default Behavior
Flux automatically checks GitHub every **5 minutes** and applies any changes. You just push to Git and forget about it!

```bash
# All you need to do:
git add .
git commit -m "Update app"
git push

# Flux handles the rest automatically within 5 minutes!
```

## Option 1: Speed Up Auto-Sync (Faster Interval)

If you want faster deployments, reduce the interval:

```bash
# Edit apps kustomization
cd ~/k8s-gitops
cat > clusters/production/apps.yaml << 'EOF'
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 1m  # Changed from 5m to 1m
  dependsOn:
    - name: infrastructure
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/production
  prune: true
EOF

git add .
git commit -m "Speed up sync to 1 minute"
git push
```

**Recommended intervals:**
- `30s` - Development (fast feedback)
- `1m` - Staging (good balance)
- `5m` - Production (current default, conservative)

## Option 2: Instant Deployment (Webhooks - RECOMMENDED!)

For instant deployments (no waiting), set up GitHub webhooks:

### Step 1: Install Flux Webhook Receiver

```bash
cd ~/k8s-gitops

# Add webhook configuration
cat > infrastructure/flux-webhook.yaml << 'EOF'
---
apiVersion: notification.toolkit.fluxcd.io/v1
kind: Receiver
metadata:
  name: github-receiver
  namespace: flux-system
spec:
  type: github
  events:
    - "ping"
    - "push"
  secretRef:
    name: webhook-token
  resources:
    - apiVersion: source.toolkit.fluxcd.io/v1
      kind: GitRepository
      name: flux-system
---
apiVersion: v1
kind: Service
metadata:
  name: webhook-receiver
  namespace: flux-system
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 9292
      protocol: TCP
  selector:
    app: notification-controller
EOF

# Generate webhook secret
WEBHOOK_TOKEN=$(head -c 12 /dev/urandom | base64 | tr -d '/+=')
kubectl create secret generic webhook-token \
  -n flux-system \
  --from-literal=token=$WEBHOOK_TOKEN

# Save token for GitHub
echo "Your webhook token: $WEBHOOK_TOKEN"
echo "Save this for GitHub webhook configuration!"

git add infrastructure/flux-webhook.yaml
git commit -m "Add webhook receiver"
git push
```

### Step 2: Get Webhook URL

```bash
# Wait for LoadBalancer IP
kubectl get svc -n flux-system webhook-receiver

# Your webhook URL will be:
# http://<LOADBALANCER-IP>/hook/<webhook-token>
```

### Step 3: Configure GitHub Webhook

1. Go to: https://github.com/jodonnell24/k8s-gitops/settings/hooks
2. Click "Add webhook"
3. Payload URL: `http://<IP>/hook/<token>` (from step 2)
4. Content type: `application/json`
5. Events: Select "Just the push event"
6. Click "Add webhook"

### Step 4: Test It!

```bash
# Make a change
cd ~/k8s-gitops
echo "# Test" >> README.md
git add . && git commit -m "Test webhook" && git push

# Deployment happens INSTANTLY (within seconds)!
# No more waiting!
```

## Current Configuration

### Apps Kustomization
```yaml
interval: 5m              # Checks every 5 minutes
path: ./apps/production
prune: true              # Removes deleted resources
dependsOn:
  - infrastructure       # Waits for infrastructure first
```

### GitRepository (Source)
```yaml
interval: 1m             # Checks Git every 1 minute
```

## Comparison

| Method | Deployment Speed | Complexity | Recommended For |
|--------|-----------------|------------|-----------------|
| **Default (5m interval)** | 0-5 minutes | Simple | Production |
| **Faster interval (1m)** | 0-1 minutes | Simple | Development |
| **Webhooks** | Instant (seconds) | Medium | All environments |

## Why Not GitHub Actions?

Flux doesn't use GitHub Actions because:

1. **Pull-based model** - Flux pulls from Git (more secure)
2. **No external triggers needed** - Works in air-gapped environments
3. **Built-in reconciliation** - Automatically fixes drift
4. **Webhooks are faster** - Instant push notifications

GitHub Actions would be **push-based** (Actions pushes to cluster), which:
- ❌ Requires cluster to be internet-accessible
- ❌ Needs credentials stored in GitHub
- ❌ Can't auto-fix manual changes (drift detection)
- ❌ Less secure

Flux's **pull-based + webhooks** model gives you:
- ✅ Instant deployments (webhooks)
- ✅ Auto-correction (polling)
- ✅ Secure (cluster pulls, not pushed to)
- ✅ Works offline/air-gapped

## Recommended Setup

**For your use case (learning/homelab):**

```yaml
# Quick feedback, simple setup
GitRepository interval: 1m
Kustomization interval: 1m
Webhooks: Optional (nice to have)
```

**For production:**

```yaml
# Conservative, with webhooks for manual deploys
GitRepository interval: 1m
Kustomization interval: 5m
Webhooks: Recommended
```

## Useful Commands

```bash
# Check when last synced
flux get kustomizations

# See sync status
kubectl get gitrepository -n flux-system
kubectl get kustomization -n flux-system

# View Flux events
kubectl get events -n flux-system --sort-by='.lastTimestamp'

# Check webhook receiver (if configured)
kubectl logs -n flux-system -l app=notification-controller --tail=50
```

## Summary

**You asked: "Do we need to flux reconcile?"**

**Answer**: NO! Flux automatically syncs every 5 minutes. You only need to:
```bash
git push   # That's it!
```

Want faster? Either:
- Reduce interval to 1m (edit `clusters/production/apps.yaml`)
- Set up webhooks (instant, recommended)

**You asked about GitHub Actions integration:**

**Answer**: Flux doesn't need it! Webhooks give you instant deployment without Actions complexity.

---

**Current Status**: Your change to 1 replica will automatically deploy within 5 minutes (or less if you set up webhooks)!

