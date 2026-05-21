# Runbook 14: Paperless-ngx

Scan, OCR, and organize all your documents.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 45 minutes |
| **Runs On** | k3s (pin to 32 GB node) or NAS |
| **Depends On** | Runbook 6 |

Deploy Paperless-ngx via Helm. The community chart bundles paperless + postgres + redis behind a single release. As with the other services, commit values to `homelab-manifests/apps/paperless/` once you're past the bootstrap.

## Step 1: Seal credentials

```bash
PAPERLESS_ADMIN_PW=$(openssl rand -base64 24)
PAPERLESS_DB_PW=$(openssl rand -base64 24)
echo "Paperless admin password: $PAPERLESS_ADMIN_PW"   # save to Vaultwarden
echo "Paperless DB password: $PAPERLESS_DB_PW"         # save to Vaultwarden

kubectl create namespace paperless

kubectl create secret generic paperless-credentials \
  --namespace paperless \
  --from-literal=admin-password="$PAPERLESS_ADMIN_PW" \
  --from-literal=db-password="$PAPERLESS_DB_PW" \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > paperless-credentials-sealed.yaml
```

## Step 2: Install via Helm

!!! warning "Chart location moves"
    The Paperless chart hosting has moved between repos. Verify the current canonical chart at [artifacthub.io](https://artifacthub.io/) (search "paperless-ngx") before pasting URLs. If the chart structure differs from these values, adapt to the chart's documented schema.

`values.yaml`:

```yaml
env:
  PAPERLESS_URL: https://paperless.yourdomain.com
  PAPERLESS_OCR_LANGUAGE: eng
  PAPERLESS_TIME_ZONE: America/New_York
  PAPERLESS_ADMIN_USER: admin
envFrom:
  - secretRef:
      name: paperless-credentials   # provides PAPERLESS_ADMIN_PASSWORD
persistence:
  data:
    enabled: true
    storageClass: nfs-storage
    size: 10Gi
  media:
    enabled: true
    storageClass: nfs-storage
    size: 50Gi
postgresql:
  enabled: true
  auth:
    existingSecret: paperless-credentials
    secretKeys:
      adminPasswordKey: db-password
redis:
  enabled: true
service:
  type: ClusterIP
nodeSelector:
  storage: large
```

Install:

```bash
helm repo add paperless-ngx https://charts.paperless-ngx.com
helm repo update
helm install paperless paperless-ngx/paperless-ngx \
  --namespace paperless \
  --values values.yaml
```

## Step 3: IngressRoute and scan source

Add an IngressRoute for `paperless.yourdomain.com` (same pattern as Vaultwarden). For scan-to-archive, mount your NAS's scan folder via a PersistentVolume backed by NFS rather than the SMB/local-path option from the original compose.

!!! tip
    Point a network scanner or phone scanning app at the consume folder via SMB on the NAS, then sync into the cluster via a NFS-backed PV. Document scanning lives in the consume folder; Paperless polls it.

## Verification

- [ ] Paperless pods Running:

    ```bash
    kubectl get pods -n paperless
    ```

- [ ] `https://paperless.yourdomain.com` loads the login screen. Login as `admin` works.
- [ ] Drop a test PDF into the NFS consume folder. Within a minute, document appears in the UI with OCR'd text.
- [ ] Postgres + Redis pods both Ready (Paperless depends on them).
