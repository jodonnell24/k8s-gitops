# Tailscale Setup Guide - Double NAT Solution

## 🎯 Why Tailscale?

Your setup has **double NAT** (ISP NAT + Home Router NAT), which makes traditional VPN impossible without cooperation from your ISP.

**Tailscale solves this:**
- ✅ No port forwarding needed
- ✅ Works through double NAT
- ✅ Built on WireGuard (same security/speed)
- ✅ Automatic NAT traversal via DERP relays
- ✅ Free for personal use (up to 100 devices)

---

## 📋 Setup Steps

### Step 1: Create Tailscale Account

1. Go to: **https://login.tailscale.com/start**
2. Sign up (free) - use Google/GitHub/Microsoft login
3. Complete account setup

### Step 2: Generate Auth Key

1. Go to: **https://login.tailscale.com/admin/settings/keys**
2. Click "**Generate auth key**"
3. Configure the key:
   - ✅ **Reusable**: Yes (can reuse if pod restarts)
   - ✅ **Ephemeral**: No (device persists)
   - ✅ **Pre-approved**: Yes (auto-approves subnet routes)
   - **Tags**: `tag:k8s` (optional, for organization)
   - **Expiration**: 90 days (or longer)
4. Click "**Generate key**"
5. **Copy the key** (starts with `tskey-auth-...`)

### Step 3: Create Sealed Secret with Auth Key

```bash
# SSH to control plane
ssh k8s-admin@10.69.69.11

# Create the secret and seal it
kubectl create secret generic tailscale-auth \
  --from-literal=AUTH_KEY='tskey-auth-YOUR-KEY-HERE' \
  --namespace=tailscale \
  --dry-run=client -o yaml | \
  kubeseal --controller-name=kube-system-sealed-secrets \
          --controller-namespace=kube-system \
          -o yaml > /tmp/tailscale-auth-sealed.yaml

# Exit and copy to GitOps repo
exit
scp k8s-admin@10.69.69.11:/tmp/tailscale-auth-sealed.yaml ~/k8s-gitops/apps/production/tailscale-auth-sealed.yaml
```

The public portfolio snapshot does not commit a live Tailscale auth secret. It keeps a non-applied example in `examples/tailscale-auth-sealed-secret.example.yaml`.

### Step 4: Update Kustomization and Deploy

```bash
cd ~/k8s-gitops

# Remove the placeholder secret from tailscale-subnet-router.yaml
# (delete the Secret section, we'll use the sealed one instead)

# Update kustomization
cat > apps/production/kustomization.yaml << 'EOF'
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - openwebui.yaml
  - tailscale-auth-sealed.yaml
  - tailscale-subnet-router.yaml
EOF

# Commit and push
git add .
git commit -m "feat: Deploy Tailscale subnet router"
git push
```

### Step 5: Approve Subnet Routes (in Tailscale Admin)

1. Go to: **https://login.tailscale.com/admin/machines**
2. Find your "**tailscale-subnet-router**" device
3. Click on it
4. Under "**Subnet routes**", click "**Edit route settings**"
5. **Approve** the routes:
   - `10.69.69.0/24` (cluster nodes)
   - `10.244.0.0/16` (pod network)
6. Click "**Save**"

### Step 6: Install Tailscale on Your Devices

**On Your Phone:**
1. Install Tailscale app from app store
2. Log in with same account
3. Connect!
4. Access services: `http://10.69.69.5` (game works!)

**On Your Laptop/PC:**
```bash
# Linux
sudo pacman -S tailscale  # Arch
sudo apt install tailscale  # Debian/Ubuntu

# Start and authenticate
sudo tailscale up
# Opens browser to log in

# macOS
brew install tailscale
sudo tailscale up

# Windows
# Download from: https://tailscale.com/download
```

---

## ✅ After Setup

When Tailscale is connected on your phone/laptop:

```
✅ Access from ANYWHERE in the world:
   http://10.69.69.5  (2048 game)
   http://10.69.69.3  (Ingress)
   ssh k8s-admin@10.69.69.11  (SSH to cluster)
   
✅ No port forwarding needed
✅ Works through double NAT
✅ Works on mobile data, coffee shop WiFi, anywhere!
✅ Automatic DNS (MagicDNS) - can enable for .k8s.lan
```

---

## 🎨 Enable MagicDNS (Optional - Makes .k8s.lan Work!)

1. Go to: https://login.tailscale.com/admin/dns
2. Enable "**MagicDNS**"
3. Add nameserver: `10.69.69.11` (your cluster)
4. Now `grafana.k8s.lan` works on all Tailscale devices!

---

## 📊 Comparison

| Feature | WireGuard (Plain) | Tailscale |
|---------|-------------------|-----------|
| Double NAT | ❌ Requires port forwarding | ✅ Works automatically |
| Setup complexity | Medium | Easy |
| Dynamic IP | ❌ Needs dynamic DNS | ✅ Handles automatically |
| Mobile app | ✅ Yes | ✅ Yes (better UX) |
| Free | ✅ Yes | ✅ Yes (personal use) |
| MagicDNS | ❌ No | ✅ Yes |

---

## 🚀 Ready to Deploy?

I can help you:
1. Set up the Tailscale account
2. Generate auth key  
3. Create sealed secret
4. Deploy via GitOps
5. Test from your phone!

Let me know when you have the auth key, or if you need help with any step!
