# k8s-gitops

This repo contains the Flux-managed Kubernetes manifests for my homelab cluster.

It includes the cluster entrypoint, shared infrastructure, and a small set of application workloads. Live secrets and generated Flux controller manifests are not committed here; the applied manifests reference Kubernetes Secrets that are created separately.

## Included

- Flux `Kustomization` resources for cluster, infrastructure, and application reconciliation.
- Cert-manager issuer and certificate manifests for internal TLS.
- CoreDNS custom records for `.k8s.lan` service names.
- Monitoring with `kube-prometheus-stack`.
- Sealed Secrets controller installation.
- Open WebUI deployment with S3-backed storage.
- Tailscale subnet router for private cluster access.

## Structure

- `clusters/production`: Flux entrypoint for the cluster.
- `infrastructure`: cluster-level services and Helm sources.
- `apps/production`: application workloads.
- `examples`: sample secret manifests used as input for Sealed Secrets or another secret manager.

## Secrets

The repo does not include live secret values. Open WebUI, Grafana, and Tailscale credentials are expected to exist in the cluster before the dependent workloads are applied.

The files in `examples/` show the expected keys and resource names.

## Applying

The cluster reconciles this repo through Flux:

```bash
flux reconcile kustomization flux-system --with-source
flux get kustomizations
```

For local validation:

```bash
kubectl kustomize clusters/production
kubectl kustomize infrastructure
kubectl kustomize apps/production
```
