# Runbook 7: Vaultwarden

Lightweight self-hosted Bitwarden-compatible server. First service runbook after Traefik — smallest dependency footprint of anything in this guide (only Traefik), and gives you a self-hosted password manager to use for every later runbook's credentials instead of pasting them into 1Password / Bitwarden Cloud.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 30 minutes |
| **Runs On** | Any node (fine on amethyst) |
| **Depends On** | Runbook 6 (HTTPS required) |

Deploy Vaultwarden via the well-maintained community Helm chart instead of docker-compose. NFS-backed persistence, ArgoCD visibility, single Traefik IngressRoute.

## Step 1: Seal the admin token

```bash
VW_ADMIN_TOKEN=$(openssl rand -base64 48)
echo "Vaultwarden admin token: $VW_ADMIN_TOKEN"   # save to Vaultwarden... your bootstrap one

kubectl create namespace vaultwarden

kubectl create secret generic vaultwarden-admin \
  --namespace vaultwarden \
  --from-literal=admin-token="$VW_ADMIN_TOKEN" \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > vaultwarden-admin-sealed.yaml

# Commit to homelab-manifests/apps/vaultwarden/
```

## Step 2: Install via Helm

`values.yaml`:

```yaml
domain: https://vault.yourdomain.com
adminToken:
  existingSecret: vaultwarden-admin
  existingSecretKey: admin-token
signupsAllowed: false
persistence:
  enabled: true
  storageClass: nfs-storage
  size: 2Gi
service:
  type: ClusterIP
```

Install:

```bash
helm repo add vaultwarden https://guerzon.github.io/vaultwarden
helm repo update
helm install vaultwarden vaultwarden/vaultwarden \
  --namespace vaultwarden \
  --values values.yaml
```

!!! warning
    Save the admin token immediately. The `/admin` panel is the only way to manage settings without DB access, and the token is the only way in.

## Step 3: IngressRoute

```yaml
# homelab-manifests/apps/vaultwarden/ingressroute.yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: vaultwarden
  namespace: vaultwarden
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`vault.yourdomain.com`)
      kind: Rule
      services:
        - name: vaultwarden
          port: 80
  tls:
    certResolver: cloudflare
    domains:
      - main: vault.yourdomain.com
```

## Step 4: Account creation (signups are off)

Because we set `signupsAllowed=false` at install, the public `/register` form is disabled. To create your first account, temporarily enable it via Helm upgrade, register, then turn it back off:

```bash
# Enable signups
helm upgrade vaultwarden vaultwarden/vaultwarden -n vaultwarden \
  --reuse-values --set signupsAllowed=true

# Visit https://vault.yourdomain.com and register your account.

# Disable signups again
helm upgrade vaultwarden vaultwarden/vaultwarden -n vaultwarden \
  --reuse-values --set signupsAllowed=false
```

!!! tip
    Vaultwarden requires HTTPS or browser extensions silently refuse to connect. The Traefik IngressRoute above handles this. If you see "cannot reach server" from the browser extension, verify the cert with `curl -v https://vault.yourdomain.com`.

## Verification

- [ ] Vaultwarden pod Running:

    ```bash
    kubectl get pods -n vaultwarden
    ```

- [ ] Browser at `https://vault.yourdomain.com` loads (no cert warning, no "cannot reach server" from extensions).
- [ ] Bitwarden/Vaultwarden browser extension connects after entering the server URL.
- [ ] `/admin` loads with the admin token from the SealedSecret.
- [ ] Signups remain disabled by default (visit `/register` — should show the disabled message).
