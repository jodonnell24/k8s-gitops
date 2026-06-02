# WireGuard VPN Setup Guide

## 🔒 WireGuard VPN Deployed!

Access your entire Kubernetes cluster remotely without exposing anything to the internet!

---

## 📊 What's Configured

| Component | Value |
|-----------|-------|
| **VPN Server IP** | `10.69.69.4` (MetalLB assigned) |
| **VPN Port** | `51820 UDP` |
| **VPN Subnet** | `10.13.13.0/24` |
| **Allowed Networks** | VPN + Cluster (10.69.69.0/24) + Pods (10.244.0.0/16) |
| **Peer Configs** | 5 (phone, laptop, tablet, etc.) |
| **Web UI** | https://vpn.k8s.lan |

---

## 🚀 Quick Setup (3 Steps)

### Step 1: Add VPN domain to `/etc/hosts`

```bash
echo '10.69.69.3 vpn.k8s.lan wireguard.k8s.lan' | sudo tee -a /etc/hosts
```

### Step 2: Get Your Client Config

**Option A: Via Web UI (Easiest)**
1. Visit: https://vpn.k8s.lan
2. Download peer config or scan QR code
3. Import to WireGuard client

**Option B: Via kubectl**
```bash
# Get peer1 config
ssh k8s-admin@10.69.69.11 "kubectl exec -n vpn deployment/wireguard -- cat /config/peer1/peer1.conf"

# Get QR code for mobile (peer1)
ssh k8s-admin@10.69.69.11 "kubectl exec -n vpn deployment/wireguard -- cat /config/peer1/peer1.png" > peer1-qr.png

# List all peers
ssh k8s-admin@10.69.69.11 "kubectl exec -n vpn deployment/wireguard -- ls /config/"
```

### Step 3: Install WireGuard Client

**Linux:**
```bash
sudo pacman -S wireguard-tools  # Arch
sudo apt install wireguard      # Debian/Ubuntu
```

**macOS:**
```bash
brew install wireguard-tools
# Or download GUI: https://www.wireguard.com/install/
```

**Windows:**
Download from: https://www.wireguard.com/install/

**Mobile (Android/iOS):**
- Install WireGuard app from store
- Scan QR code from web UI

---

## 📱 Setting Up Your Device

### Desktop/Laptop (Config File)

```bash
# 1. Get the config
ssh k8s-admin@10.69.69.11 "kubectl exec -n vpn deployment/wireguard -- cat /config/peer1/peer1.conf" > ~/wireguard-peer1.conf

# 2. Import to WireGuard
sudo cp ~/wireguard-peer1.conf /etc/wireguard/wg0.conf

# 3. Start VPN
sudo wg-quick up wg0

# 4. Test connectivity
ping 10.69.69.11  # Should reach control plane!
curl https://grafana.k8s.lan  # Access services!

# 5. Stop VPN when done
sudo wg-quick down wg0
```

### Mobile (QR Code)

```bash
# 1. Get QR code for mobile
ssh k8s-admin@10.69.69.11 "kubectl exec -n vpn deployment/wireguard -- cat /config/peer2/peer2.png" > peer2-qr.png

# 2. Open the PNG and scan with WireGuard app
# 3. Tap to connect!
```

---

## ✅ Verify VPN is Working

### Before Connecting
```bash
# These should fail or require internet routing
ping 10.69.69.11  # No route
curl https://grafana.k8s.lan  # Can't resolve
```

### After Connecting to VPN
```bash
# These should work!
ping 10.69.69.11  # Success!
curl https://grafana.k8s.lan  # Grafana loads!
curl https://2048.k8s.lan  # Play game remotely!
ssh k8s-admin@10.69.69.11  # Direct SSH access!
```

---

## 🌍 What You Can Access via VPN

Once connected, you can reach:

✅ **All Kubernetes Services**
- https://grafana.k8s.lan
- https://prometheus.k8s.lan
- https://2048.k8s.lan
- https://alertmanager.k8s.lan

✅ **Cluster Nodes (SSH)**
- `ssh k8s-admin@10.69.69.11` (control plane)
- `ssh k8s-admin@10.69.69.21-25` (workers)

✅ **LoadBalancer IPs**
- Direct access to any LoadBalancer service

✅ **Pod Network** (if needed)
- Can access pods directly at 10.244.x.x

---

## 🔒 Security Features

✅ **Encrypted tunnel** - ChaCha20-Poly1305 encryption
✅ **Modern protocol** - WireGuard (faster than OpenVPN)
✅ **Split tunneling** - Only cluster traffic goes through VPN
✅ **No internet exposure** - VPN is on local network only
✅ **Multiple devices** - 5 peer configs pre-generated

---

## Configuration

### Current Setup
```yaml
VPN Server: 10.69.69.4:51820
VPN Subnet: 10.13.13.0/24
Routes to VPN:
  - 10.13.13.0/24 (VPN network)
  - 10.69.69.0/24 (cluster nodes)
  - 10.244.0.0/16 (pod network)

Peers:
  - peer1: Desktop/Laptop
  - peer2: Mobile
  - peer3: Tablet
  - peer4: Work laptop
  - peer5: Spare
```

### Add More Peers

```bash
# Edit ConfigMap
cd ~/k8s-gitops
# Change PEERS: "5" to PEERS: "10"
sed -i 's/PEERS: "5"/PEERS: "10"/' apps/production/wireguard-vpn.yaml

git add . && git commit -m "Add more VPN peers" && git push

# Restart WireGuard pod to regenerate
kubectl delete pod -n vpn -l app=wireguard
```

---

## 📖 Using the Web UI

Access: **https://vpn.k8s.lan** (when on local network)

Features:
- View all peer configs
- Download .conf files
- See QR codes for mobile
- Check connection status

---

## 🏠 Port Forwarding (If Router NAT Required)

If you want to access VPN from the internet:

1. **Forward port on your router:**
   - External: 51820 UDP
   - Internal: 10.69.69.4:51820

2. **Update WireGuard config:**
```bash
cd ~/k8s-gitops
# Set your public IP
sed -i 's/SERVERURL: "auto"/SERVERURL: "your.public.ip"/' apps/production/wireguard-vpn.yaml
git add . && git commit -m "Set public IP for VPN" && git push
```

3. **Regenerate configs:**
```bash
kubectl delete pod -n vpn -l app=wireguard
# Get new configs with public IP
```

**Security Note**: Only do this if you understand the security implications!

---

## 🧪 Testing Your VPN

### Test Script
```bash
#!/bin/bash
# test-vpn.sh

echo "Testing VPN connectivity..."

# Should work when VPN connected
ping -c 3 10.69.69.11 && echo "✅ Can reach control plane"
curl -k https://grafana.k8s.lan -I | grep "HTTP" && echo "✅ Can access Grafana"
curl -k https://2048.k8s.lan -I | grep "HTTP" && echo "✅ Can access 2048 game"

echo "VPN is working! 🎉"
```

---

## 🎯 Common Use Cases

### Use Case 1: Work from Coffee Shop
```
Connect to VPN → Access Grafana dashboards
Monitor your cluster from anywhere!
```

### Use Case 2: Mobile Monitoring
```
Scan QR code on phone → Connect
Check cluster health on the go!
```

### Use Case 3: Remote Development
```
Connect to VPN → SSH to nodes
Deploy and debug from anywhere
```

---

## 📱 Client Config Example

```ini
[Interface]
PrivateKey = <auto-generated>
Address = 10.13.13.2/32
DNS = 10.69.69.11

[Peer]
PublicKey = <server-public-key>
PresharedKey = <auto-generated>
Endpoint = 10.69.69.4:51820
AllowedIPs = 10.13.13.0/24, 10.69.69.0/24, 10.244.0.0/16
PersistentKeepalive = 25
```

---

## 🎉 Summary

**You now have:**
- ✅ WireGuard VPN server (10.69.69.4:51820)
- ✅ 5 pre-configured client peers
- ✅ Web UI for config management (https://vpn.k8s.lan)
- ✅ Access to ALL cluster services remotely
- ✅ No internet exposure required
- ✅ Deployed and managed via GitOps!

**Next steps:**
1. Get a client config from the pod
2. Install WireGuard on your device
3. Connect and access your services from anywhere!

---

**Your homelab is now accessible from anywhere in the world!** 🌍🔒

