# Authelia SSO

Authentication proxy for services on minhnguyenle.net.

## Setup

1. Generate a password hash:
```bash
docker run --rm authelia/authelia:latest authelia crypto hash generate argon2 --password 'your-password'
```

2. Copy `users.yml.example` to `users.yml` and replace the hash.

3. Apply:
```bash
kubectl create secret generic authelia-users \
  --namespace=authelia \
  --from-file=users.yml=users.yml \
  --dry-run=client -o yaml | kubectl apply -f -
```

## Protect a new service

Add the domain to `configuration.yml` under `access_control.rules`, then add this middleware to the service's IngressRoute:
```yaml
middlewares:
  - name: authelia-forwardauth
```
