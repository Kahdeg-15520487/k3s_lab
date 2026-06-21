# Homelab Kubernetes — Kustomize Overlays

Infrastructure-as-code for a bare-metal Kubernetes homelab, managed via Kustomize.
Apply each overlay against the live cluster with `kubectl apply -k`.

## Structure

```
├── cert-manager/       # 2-tier internal CA (root → intermediate)
├── metallb/            # L2 load balancer, IP pool 192.168.1.220-240
├── metrics-server/     # Metrics server with tolerations & custom args
├── headlamp/           # Kubernetes web UI + cluster-admin SA + Traefik TLS ingress
├── traefik-dashboard/  # Traefik dashboard with BasicAuth + ingress route
```

## Deployment order

1. **cert-manager** — issuers must exist before any Certificate resources are applied
2. **metallb** — load balancer IPs for Service type LoadBalancer
3. **metrics-server** — node metrics collection
4. **headlamp** — depends on cert-manager (TLS) and Traefik CRDs
5. **traefik-dashboard** — depends on Traefik CRDs

```bash
kubectl apply -k cert-manager/deployment/
kubectl apply -k cert-manager/certificates/
kubectl apply -k metallb/
kubectl apply -k metrics-server/
kubectl apply -k headlamp/
kubectl apply -k traefik-dashboard/
```

## Prerequisites

- Traefik installed as the ingress controller with `websecure` entrypoint
- cert-manager CRDs installed
- MetalLB running in `metallb-system` namespace

## Secrets

### Traefik dashboard BasicAuth

The `traefik-dashboard` secret is **not** committed to git.

```bash
# Generate the bcrypt hash
htpasswd -nbB tfkuser <your-password> > traefik-dashboard/secrets/users

# Or manually create the file with format: username:bcrypt_hash
echo "tfkuser:$(htpasswd -nbB tfkuser <your-password> | cut -d: -f2-)" > traefik-dashboard/secrets/users
```

See `traefik-dashboard/secrets/users.example` for the expected format.
