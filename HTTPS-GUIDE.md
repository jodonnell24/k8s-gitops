## 🔒 HTTPS Setup Guide

Your cluster now has HTTPS fully configured!

### What's Configured

- ✅ Cert-Manager with self-signed CA
- ✅ Automatic certificate issuance
- ✅ Force HTTPS redirect (HTTP → HTTPS)
- ✅ HSTS headers for security
- ✅ HTTP/2 support

### Quick Access

1. **Add to `/etc/hosts`:**
   ```bash
   echo "10.69.69.3 whoami.k8s.lan" | sudo tee -a /etc/hosts
   ```

2. **Access in browser:**
   ```
   https://whoami.k8s.lan
   ```

3. **Trust the CA (no warnings):**
   ```bash
   ~/trust-k8s-ca.sh
   ```
   Then restart your browser.

### Adding HTTPS to New Apps

When deploying new apps, just add these sections:

```yaml
---
# Certificate resource
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-tls
  namespace: demo
spec:
  secretName: myapp-tls-secret
  issuerRef:
    name: ca-issuer
    kind: ClusterIssuer
  dnsNames:
    - myapp.k8s.lan
---
# Ingress with TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: demo
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.k8s.lan
    secretName: myapp-tls-secret
  rules:
  - host: myapp.k8s.lan
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
```

### Certificate Verification

```bash
# Check certificates
kubectl get certificates -A

# Check certificate details
kubectl describe certificate whoami-tls -n demo

# View certificate secret
kubectl get secret whoami-tls-secret -n demo -o yaml
```

### Production Options (Future)

For production environments, you can switch to:

1. **Let's Encrypt** - Free, trusted certificates
   - Requires public domain
   - DNS or HTTP challenge

2. **Corporate CA** - Use your company's CA
   - Import existing CA certificate
   - All certs signed by corporate CA

3. **Cloud Provider Certs** - AWS ACM, etc.
   - External certificate management
   - Integration with cloud load balancers

### Security Features Enabled

- ✅ **TLS 1.2+** - Modern encryption only
- ✅ **HTTP/2** - Faster, more efficient
- ✅ **HSTS** - Force HTTPS for 1 year
- ✅ **Auto-redirect** - HTTP → HTTPS (308)
- ✅ **Certificate renewal** - Auto-renewed by cert-manager

Your cluster is now following HTTPS best practices! 🔒

