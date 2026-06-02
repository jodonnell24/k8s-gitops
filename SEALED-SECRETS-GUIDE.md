# Phase 4: Sealed Secrets - Complete Guide

## 🔐 Sealed Secrets Deployed!

You can now safely store encrypted secrets in Git!

---

## How Sealed Secrets Work

```
┌─────────────────────────────────────────────────────────────┐
│ Traditional Secrets (❌ UNSAFE)                             │
├─────────────────────────────────────────────────────────────┤
│ Secret (plain text) → Git → ❌ EXPOSED!                    │
│ Anyone with Git access sees passwords                       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Sealed Secrets (✅ SAFE)                                    │
├─────────────────────────────────────────────────────────────┤
│ Secret (plain) → kubeseal → SealedSecret (encrypted)       │
│                  ↓                                           │
│                 Git → Flux → Cluster                        │
│                            ↓                                 │
│              Sealed Secrets Controller                      │
│                            ↓                                 │
│                  Secret (decrypted in cluster only!)        │
└─────────────────────────────────────────────────────────────┘
```

**Key Point**: Only the cluster can decrypt! Even if someone steals your Git repo, they can't read the secrets.

---

## Quick Start: Creating a Sealed Secret

### Method 1: From Control Plane (Easiest)

```bash
# SSH to control plane
ssh k8s-admin@10.69.69.11

# Create and seal a secret
kubectl create secret generic my-app-secret \
  --from-literal=username=admin \
  --from-literal=password=supersecret123 \
  --namespace=games \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=kube-system-sealed-secrets \
    --controller-namespace=kube-system \
    -o yaml > /tmp/my-sealed-secret.yaml

# Copy to your workstation
exit
scp k8s-admin@10.69.69.11:/tmp/my-sealed-secret.yaml ~/k8s-gitops/apps/production/

# Commit and push
cd ~/k8s-gitops
git add apps/production/my-sealed-secret.yaml
git commit -m "Add sealed secret"
git push

# Flux automatically applies it!
# The secret is decrypted in cluster automatically
```

### Method 2: From Your Local PC (After Installing kubeseal)

```bash
# Install kubeseal on your PC
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.2/kubeseal-0.27.2-linux-amd64.tar.gz
tar xfz kubeseal-0.27.2-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

# Fetch the public key from cluster (one-time)
kubeseal --fetch-cert \
  --controller-name=kube-system-sealed-secrets \
  --controller-namespace=kube-system \
  > ~/.sealed-secrets-pub.pem

# Now you can seal secrets locally
kubectl create secret generic my-secret \
  --from-literal=password=secret123 \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=kube-system-sealed-secrets \
    --controller-namespace=kube-system \
    --cert ~/.sealed-secrets-pub.pem -o yaml \
  > apps/production/my-sealed-secret.yaml

# Commit and push!
git add . && git commit -m "Add secret" && git push
```

---

## Example: Public-Safe Sealed Secret

This public snapshot keeps examples under `examples/` instead of committing live cluster secrets:

```yaml
# examples/tailscale-auth-sealed-secret.example.yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: tailscale-auth
  namespace: tailscale
spec:
  encryptedData:
    AUTH_KEY: AgA_REPLACE_WITH_KUBESEAL_OUTPUT
```

Real SealedSecret output is safe to commit only when it was generated for the intended cluster and the underlying credential can be rotated.

---

## Using Sealed Secrets in Your Apps

Once the SealedSecret is applied, it creates a regular Secret. Use it normally:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-app:latest
        env:
          # Reference the decrypted secret
          - name: API_KEY
            valueFrom:
              secretKeyRef:
                name: game-config  # Name from SealedSecret
                key: api-key
          - name: DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name: game-config
                key: db-password
        # Or mount as files
        volumeMounts:
        - name: secrets
          mountPath: /secrets
          readOnly: true
      volumes:
      - name: secrets
        secret:
          secretName: game-config
```

---

## Verify Sealed Secrets Work

```bash
# Check sealed secret was created
kubectl get sealedsecrets -n games

# Check regular secret was auto-created
kubectl get secret game-config -n games

# View the decrypted values (only works in cluster!)
kubectl get secret game-config -n games -o jsonpath='{.data.api-key}' | base64 -d
# Output: super-secret-key-12345
```

---

## Common Operations

### Create a sealed secret from file
```bash
# SSH to control plane
ssh k8s-admin@10.69.69.11

# Create secret from file
kubectl create secret generic my-files \
  --from-file=config.json \
  --from-file=key.pem \
  --namespace=games \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=kube-system-sealed-secrets \
    --controller-namespace=kube-system \
    -o yaml > /tmp/my-files-sealed.yaml

# Copy to your workstation
exit
scp k8s-admin@10.69.69.11:/tmp/my-files-sealed.yaml ~/k8s-gitops/apps/production/
```

### Create a TLS secret
```bash
# SSH to control plane
ssh k8s-admin@10.69.69.11

# Create TLS secret
kubectl create secret tls my-tls \
  --cert=path/to/cert.crt \
  --key=path/to/key.key \
  --namespace=games \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=kube-system-sealed-secrets \
    --controller-namespace=kube-system \
    -o yaml > /tmp/my-tls-sealed.yaml

# Copy to your workstation
exit
scp k8s-admin@10.69.69.11:/tmp/my-tls-sealed.yaml ~/k8s-gitops/apps/production/
```

### Update a sealed secret
```bash
# SSH to control plane
ssh k8s-admin@10.69.69.11

# Create new version
kubectl create secret generic my-secret \
  --from-literal=password=NEW-PASSWORD \
  --namespace=games \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=kube-system-sealed-secrets \
    --controller-namespace=kube-system \
    -o yaml > /tmp/my-sealed-secret.yaml

# Copy to your workstation
exit
scp k8s-admin@10.69.69.11:/tmp/my-sealed-secret.yaml ~/k8s-gitops/apps/production/

# Commit and push - replaces old one
git add . && git commit -m "Update secret" && git push
```

### Delete a sealed secret
```bash
# Remove from Git
rm apps/production/my-sealed-secret.yaml
git add . && git commit -m "Remove secret" && git push

# Flux automatically deletes it from cluster!
```

---

## Backup & Disaster Recovery

### Backup the Sealing Key (IMPORTANT!)

```bash
# Export the master key
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > sealed-secrets-master-key.yaml

# Store this file SECURELY (NOT in Git!)
# - Encrypted USB drive
# - Password manager
# - Encrypted backup

# To restore on new cluster:
kubectl apply -f sealed-secrets-master-key.yaml
kubectl delete pod -n kube-system -l app.kubernetes.io/name=sealed-secrets
```

**Why backup?** If you lose the key and rebuild your cluster, you can't decrypt your sealed secrets!

---

## Security Best Practices

✅ **DO:**
- Commit SealedSecrets to Git (they're encrypted)
- Use different namespaces for different apps
- Backup the sealing key securely
- Rotate secrets regularly

❌ **DON'T:**
- Commit regular Secrets to Git
- Share the sealing key publicly
- Keep plain-text secrets on your workstation
- Reuse secrets across environments

---

## Troubleshooting

### Secret not decrypting
```bash
# Check sealed secrets controller logs
kubectl logs -n kube-system -l app.kubernetes.io/name=sealed-secrets

# Check sealed secret status
kubectl describe sealedsecret game-config -n games

# Common issue: Wrong namespace
# Sealed secrets are namespace-scoped by default
```

### Can't seal secrets from local PC
```bash
# Make sure you fetched the public cert
kubeseal --fetch-cert \
  --controller-name=kube-system-sealed-secrets \
  --controller-namespace=kube-system \
  > ~/.sealed-secrets-pub.pem

# Test it
echo "test" | kubeseal --raw --cert ~/.sealed-secrets-pub.pem
```

---

## Phase 4 Complete!

You now have:
- ✅ Sealed Secrets controller running
- ✅ kubeseal CLI installed (control plane)
- ✅ Example sealed secret created
- ✅ Safe to commit secrets to Git
- ✅ Automatic decryption in cluster

---

## 🎯 **ALL PHASES COMPLETE!**

✅ **Phase 1**: Core Infrastructure  
✅ **Phase 2**: GitOps with Flux  
✅ **Phase 3**: Observability  
✅ **Phase 4**: Secrets Management  

**Your cluster is PRODUCTION-READY!** 🚀

---

## Quick Reference Card

```bash
# Seal a secret (on control plane)
ssh k8s-admin@10.69.69.11
kubectl create secret generic NAME --from-literal=KEY=VALUE --namespace=NS \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=kube-system-sealed-secrets \
    --controller-namespace=kube-system \
    -o yaml > /tmp/sealed.yaml

# Copy to GitOps repo
exit
scp k8s-admin@10.69.69.11:/tmp/sealed.yaml ~/k8s-gitops/apps/production/

# Deploy via Git
cd ~/k8s-gitops
git add . && git commit -m "Add secret" && git push

# Verify
kubectl get sealedsecret -n NS
kubectl get secret -n NS
```

---

**Next**: Deploy more apps with secrets! Database credentials, API keys, etc.
