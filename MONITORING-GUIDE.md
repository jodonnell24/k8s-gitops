# Phase 3: Monitoring - Quick Start Guide

## 🎉 Monitoring Stack Deployed!

Your cluster now has full observability with Prometheus and Grafana!

---

## 🌐 Access Your Dashboards

### Step 1: Add to `/etc/hosts` (on your PC)
```bash
echo '10.69.69.3 grafana.k8s.lan' | sudo tee -a /etc/hosts
echo '10.69.69.3 prometheus.k8s.lan' | sudo tee -a /etc/hosts
echo '10.69.69.3 alertmanager.k8s.lan' | sudo tee -a /etc/hosts
```

### Step 2: Access in Browser

**Grafana** (Main Dashboard)
- URL: **https://grafana.k8s.lan**
- Username: `admin`
- Password: `admin`
- Features: Pre-built dashboards, beautiful visualizations

**Prometheus** (Raw Metrics)
- URL: **https://prometheus.k8s.lan**
- Direct access to metrics and queries
- PromQL query interface

**AlertManager** (Alert Management)
- URL: **https://alertmanager.k8s.lan**
- Manages and routes alerts
- Notification configuration

---

## 📊 What's Monitoring

### Cluster Metrics (Automatic)
- ✅ Node CPU, memory, disk usage
- ✅ Pod resource usage
- ✅ Network traffic
- ✅ Container metrics

### Kubernetes Objects (Automatic)
- ✅ Deployments, pods, services
- ✅ PersistentVolumeClaims
- ✅ Ingresses, nodes
- ✅ Resource quotas

### Application Metrics (Coming Soon)
- Configure your apps to expose metrics
- Prometheus automatically discovers and scrapes them

---

## 🎨 Grafana Pre-built Dashboards

Once logged in to Grafana, check out these dashboards:

1. **Kubernetes / Compute Resources / Cluster**
   - Overall cluster CPU/memory
   - Node resource usage
   - Pod distribution

2. **Kubernetes / Compute Resources / Namespace (Pods)**
   - Per-namespace resource usage
   - Pod CPU/memory graphs

3. **Kubernetes / Compute Resources / Node (Pods)**
   - Per-node metrics
   - Pod capacity

4. **Node Exporter / Nodes**
   - Detailed node metrics
   - CPU, memory, disk, network

---

## 🔍 Quick Grafana Tour

### First Login
1. Go to https://grafana.k8s.lan
2. Login: `admin` / `admin`
3. You'll be prompted to change password (optional for homelab)

### Navigate Dashboards
1. Click ☰ menu (top left)
2. Click "Dashboards"
3. Browse available dashboards
4. Try "Kubernetes / Compute Resources / Cluster"

### Explore Metrics
1. Click ☰ menu → "Explore"
2. Select "Prometheus" as data source
3. Try query: `node_cpu_seconds_total`
4. Click "Run query"

---

## 📈 Useful Prometheus Queries

Try these in Grafana Explore or Prometheus UI:

```promql
# CPU usage by node
rate(node_cpu_seconds_total{mode="idle"}[5m])

# Memory usage
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes

# Pod count by namespace
count(kube_pod_info) by (namespace)

# Network traffic
rate(node_network_receive_bytes_total[5m])

# Container restarts
kube_pod_container_status_restarts_total

# Disk usage
(node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes
```

---

## ⚙️ Configuration

### Current Settings
- **Prometheus retention**: 7 days
- **Storage**: emptyDir (no persistence)
- **Grafana password**: `admin` (change in production!)
- **HTTPS**: Enabled on all services
- **Resources**: Reasonable limits for homelab

### Customize (via GitOps)

Edit `infrastructure/monitoring/kube-prometheus-stack.yaml` and push:

```bash
cd ~/k8s-gitops

# Example: Change Grafana password
# Edit the file, change adminPassword value
nano infrastructure/monitoring/kube-prometheus-stack.yaml

git add .
git commit -m "Update Grafana password"
git push

# Flux automatically applies the change!
```

---

## 🚨 Alerting (Optional)

### View Alerts
- Go to Grafana → Alerting → Alert rules
- Or visit: https://alertmanager.k8s.lan

### Configure Notifications
Edit alert rules and notification channels in:
- `infrastructure/monitoring/kube-prometheus-stack.yaml`
- Add Slack, email, or other integrations

---

## 🧪 Test the Monitoring

### Create Load
```bash
# Deploy a CPU-intensive app
kubectl run stress --image=polinux/stress --command -- stress --cpu 2

# Watch in Grafana
# Go to: Kubernetes / Compute Resources / Cluster
# See CPU spike in real-time!

# Cleanup
kubectl delete pod stress
```

### Check Your Apps
```bash
# View whoami pod metrics
# In Grafana Explore:
container_cpu_usage_seconds_total{namespace="demo", pod=~"whoami.*"}
```

---

## 📦 What's Deployed

| Component | Replicas | Purpose |
|-----------|----------|---------|
| Prometheus | 1 | Metrics storage & queries |
| Grafana | 1 | Visualization dashboards |
| AlertManager | 1 | Alert routing |
| Node Exporter | 6 | Node metrics (1 per node) |
| Kube State Metrics | 1 | K8s object metrics |
| Prometheus Operator | 1 | Manages Prometheus CRDs |

---

## 💾 Note About Persistence

**Current mode: No persistence (emptyDir)**
- Metrics stored in pod memory/disk
- **Data lost on pod restart**
- Fine for learning/homelab

**Future: Add persistence**
- Install local-path-provisioner or Longhorn
- Enable `persistence` in Helm values
- Metrics survive pod restarts

For now, keeping it simple! Your dashboards and queries still work perfectly.

---

## 🎯 Phase 3 Complete!

You now have:
- ✅ Full cluster observability
- ✅ Beautiful Grafana dashboards
- ✅ Prometheus metrics collection
- ✅ All deployed via GitOps!
- ✅ HTTPS on all monitoring services

Next: Phase 4 (Sealed Secrets) or start deploying your own apps!

---

**Access Grafana now**: https://grafana.k8s.lan (admin/admin) 🎨

