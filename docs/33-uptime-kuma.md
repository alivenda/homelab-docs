# Runbook 33: Uptime Kuma

Self-hosted uptime monitoring with status pages and ntfy alerts.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 20–30 minutes |
| **Runs On** | k3s (cluster) |
| **Depends On** | Runbook 5 (k3s), Runbook 6 (Traefik), Runbook 18 (Authelia — ForwardAuth), Runbook 19 (ntfy) |

Uptime Kuma ([uptime-kuma.app](https://uptime-kuma.app)) monitors your services, sends alerts via ntfy, and generates public status pages. ARM64 ✅ — verify current ARM64 support on [Docker Hub](https://hub.docker.com/r/louislam/uptime-kuma). See the [wiki](https://github.com/louislam/uptime-kuma/wiki) for reference.

**Auth mode:** Authelia ForwardAuth. Uptime Kuma has its own login — ForwardAuth adds a second layer in front.

!!! warning "Do not use NFS storage for Uptime Kuma"
    Uptime Kuma uses SQLite, which requires POSIX file locking. NFS does not support POSIX locks reliably and will corrupt the database. This runbook uses k3s's built-in `local-path` StorageClass, which creates node-local storage with proper filesystem semantics.

## Step 1: Deploy Uptime Kuma

Create `homelab-manifests/apps/uptime-kuma/deployment.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: uptime-kuma
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: uptime-kuma
  namespace: uptime-kuma
spec:
  replicas: 1
  selector:
    matchLabels:
      app: uptime-kuma
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: uptime-kuma
    spec:
      containers:
        - name: uptime-kuma
          image: louislam/uptime-kuma:2
          ports:
            - containerPort: 3001
          volumeMounts:
            - name: data
              mountPath: /app/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: uptime-kuma-data
```

!!! note "Recreate strategy"
    `Recreate` (not `RollingUpdate`) is required because the local-path PVC can only be mounted on one node at a time. A rolling update would attempt to start a second pod before the first releases the volume.

Create `homelab-manifests/apps/uptime-kuma/pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: uptime-kuma-data
  namespace: uptime-kuma
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi
```

`local-path` is the k3s default StorageClass and creates node-local storage. The SQLite database will be stored on whichever cluster node the pod schedules to.

Create `homelab-manifests/apps/uptime-kuma/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: uptime-kuma
  namespace: uptime-kuma
spec:
  selector:
    app: uptime-kuma
  ports:
    - port: 3001
      targetPort: 3001
```

## Step 2: IngressRoute

Create `homelab-manifests/apps/uptime-kuma/ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: uptime-kuma
  namespace: uptime-kuma
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`status.yourdomain.com`)
      kind: Rule
      middlewares:
        - name: authelia@kubernetescrd
          namespace: authelia
      services:
        - name: uptime-kuma
          port: 3001
  tls:
    certResolver: cloudflare
    domains:
      - main: status.yourdomain.com
```

Commit all manifests to `homelab-manifests/apps/uptime-kuma/` and let ArgoCD sync.

## Step 3: Initial Setup

Open `https://status.yourdomain.com` (log in with Authelia first). The Uptime Kuma setup wizard asks for a username and password — set a strong one and save to Vaultwarden.

## Step 4: Add Monitors

In the Uptime Kuma dashboard, add monitors for every service in the stack:

| Monitor name | URL | Type |
|---|---|---|
| Homepage | `https://homepage.yourdomain.com` | HTTP |
| Authelia | `https://auth.yourdomain.com` | HTTP |
| Nextcloud | `https://cloud.yourdomain.com` | HTTP |
| Immich | `https://photos.yourdomain.com` | HTTP |
| Forgejo | `https://git.yourdomain.com` | HTTP |
| Vaultwarden | `https://vault.yourdomain.com` | HTTP |
| ArgoCD | `https://argocd.yourdomain.com` | HTTP |
| Grafana | `https://grafana.yourdomain.com` | HTTP |

Set the check interval to 60 seconds for each.

## Step 5: Configure ntfy Notifications

1. Go to **Settings → Notifications → Add Notification**.
2. Select **ntfy** as the notification type.
3. Set:
   - **Server URL**: `https://ntfy.yourdomain.com`
   - **Topic**: `homelab-alerts`
   - **Username / Password**: your ntfy credentials from Vaultwarden
4. Click **Test** to verify a notification arrives on your phone.
5. Apply the notification to all monitors.

## Verification

- [ ] Uptime Kuma pod Running:

    ```bash
    kubectl get pods -n uptime-kuma
    ```

- [ ] `https://status.yourdomain.com` loads Uptime Kuma after Authelia login.
- [ ] All monitors show green.
- [ ] Test notification arrives in ntfy mobile app.
- [ ] Stop a test service and confirm a down alert fires within 2 minutes.
- [ ] Uptime Kuma credentials saved to Vaultwarden.
