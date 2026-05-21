# Runbook 12: Nextcloud

Self-hosted file sync, calendar, contacts.

| | |
|---|---|
| **Difficulty** | Beginner–Intermediate |
| **Time Estimate** | 1–2 hours |
| **Runs On** | k3s (pin to 32 GB node) |
| **Depends On** | Runbook 5, Runbook 6 |

## Install

```bash
helm repo add nextcloud https://nextcloud.github.io/helm/

kubectl create namespace nextcloud

helm install nextcloud nextcloud/nextcloud \
  --namespace nextcloud \
  --set nextcloud.host=nextcloud.yourdomain.com \
  --set persistence.enabled=true \
  --set persistence.storageClass=nfs-storage \
  --set persistence.size=100Gi \
  --set internalDatabase.enabled=false \
  --set externalDatabase.type=postgresql \
  --set postgresql.enabled=true \
  --set service.type=ClusterIP \
  --set nodeSelector.storage=large
```

!!! tip
    Cron-based background jobs are dramatically more reliable than AJAX. Configure under **Settings → Basic Settings**.

## IngressRoute

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: nextcloud
  namespace: nextcloud
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`nextcloud.yourdomain.com`)
      kind: Rule
      services:
        - name: nextcloud
          port: 80
  tls:
    certResolver: cloudflare
    domains:
      - main: nextcloud.yourdomain.com
```

## Verification

- [ ] All Nextcloud pods Running:

    ```bash
    kubectl get pods -n nextcloud
    ```

- [ ] `https://nextcloud.yourdomain.com` loads the setup wizard or login.
- [ ] After admin setup, sync a file via the desktop client. File appears on another device within ~1 minute.
- [ ] Cron job runs (Settings → Basic Settings → confirm Last cron job was recent).
