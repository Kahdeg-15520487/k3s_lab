# Homelab Kubernetes — Kustomize Overlays

Infrastructure-as-code for a bare-metal Kubernetes homelab, managed via Kustomize.
Apply each overlay against the live cluster with `kubectl apply -k`.

## Structure

```
├── cert-manager/
│   ├── deployment/     # cert-manager itself (upstream manifest)
│   ├── certificates/   # 2-tier internal CA (root → intermediate)
│   └── acme/           # Let's Encrypt ClusterIssuer + Cloudflare DNS-01
├── letsencrypt/        # Wildcard cert (*.minhnguyenle.net) + Traefik default TLS
├── metallb/            # L2 load balancer, IP pool 192.168.1.220-240
├── metrics-server/     # Metrics server with tolerations & custom args
├── ddns/               # Cloudflare DDNS CronJob — updates A record when ISP IP changes
├── headlamp/           # Kubernetes web UI + cluster-admin SA + Traefik TLS ingress
└── traefik-dashboard/  # Traefik dashboard with BasicAuth + ingress route
```

## Deployment order

```bash
# 1. cert-manager (base)
kubectl apply -k cert-manager/deployment/

# 2. Let's Encrypt ACME (Cloudflare DNS-01)
#    First, create the secret file from the template:
#    cp cert-manager/acme/cloudflare-api-token-secret.env.example cert-manager/acme/cloudflare-api-token-secret.env
#    Edit it with your real Cloudflare API token (Zone:DNS:Edit on your domain).
kubectl apply -k cert-manager/acme/

# 3. Internal CA (for cluster-internal services)
kubectl apply -k cert-manager/certificates/

# 4. Wildcard certificate + Traefik default TLS
kubectl apply -k letsencrypt/

# 5. DDNS (keeps *.minhnguyenle.net pointing to your public IP)
#    Create the token file same as step 2:
#    cp ddns/cloudflare-ddns-api-token.env.example ddns/cloudflare-ddns-api-token.env
#    Edit with your Cloudflare API token.
kubectl apply -k ddns/

# 6. Infrastructure services
kubectl apply -k metallb/
kubectl apply -k metrics-server/
kubectl apply -k headlamp/
kubectl apply -k traefik-dashboard/
```

## Prerequisites

- Traefik installed as the ingress controller with `websecure` entrypoint
- cert-manager CRDs installed
- MetalLB running in `metallb-system` namespace
- **Port forwarding** on your router: TCP 80 → `192.168.1.220:80`, TCP 443 → `192.168.1.220:443`
- Cloudflare API token with Zone:DNS:Edit on your domain

## Certificate architecture

```
Let's Encrypt (staging)
  └── ClusterIssuer: letsencrypt-staging (DNS-01 via Cloudflare)
       └── Certificate: *.minhnguyenle.net → secret: wildcard-minhnguyenle-net-tls
            └── TLSStore (default) → all websecure IngressRoutes inherit it

Internal CA (homelab.cloud)
  └── Root CA (self-signed) → Intermediate CA
       └── Per-service certs (cluster-internal use)
```

All services use hostnames under `*.minhnguyenle.net`:

| Service | URL |
|---|---|
| Headlamp | `https://headlamp.minhnguyenle.net` |
| Traefik Dashboard | `https://traefik.minhnguyenle.net` |

## Secrets

### Cloudflare API token (Let's Encrypt DNS-01)

The token file is **not** committed to git.

```bash
# Create from template
cp cert-manager/acme/cloudflare-api-token-secret.env.example cert-manager/acme/cloudflare-api-token-secret.env
# Edit: replace YOUR_CLOUDFLARE_API_TOKEN_HERE with a real token
# Token needs: Zone — DNS — Edit on minhnguyenle.net
# Create at: https://dash.cloudflare.com/profile/api-tokens
```

### Traefik dashboard BasicAuth

The `traefik-dashboard` secret is **not** committed to git.

```bash
htpasswd -nbB tfkuser <your-password> > traefik-dashboard/secrets/users
```

See `traefik-dashboard/secrets/users.example` for the expected format.

### Cloudflare API token (DDNS)

Same token as cert-manager ACME. The DDNS CronJob updates the `*.minhnguyenle.net` A record every 5 minutes.

```bash
cp ddns/cloudflare-ddns-api-token.env.example ddns/cloudflare-ddns-api-token.env
# Edit with your Cloudflare API token
```
