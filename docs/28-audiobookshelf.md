# Runbook 28: Audiobookshelf

Self-hosted audiobook and podcast server with progress sync and mobile apps.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 30–45 minutes |
| **Runs On** | k3s (cluster) — library on NAS via NFS |
| **Depends On** | Runbook 5 (k3s), Runbook 6 (Traefik), Runbook 18 (Authelia — ForwardAuth) |

Audiobookshelf ([audiobookshelf.org](https://www.audiobookshelf.org)) is a self-hosted audiobook and podcast server with dedicated Android and iOS apps, chapter support, and listen progress sync. ARM64 ✅ (`ghcr.io/advplyr/audiobookshelf` ships multiarch). See the [Docker install docs](https://www.audiobookshelf.org/docs#docker-install) for full reference.

**Auth mode:** Authelia ForwardAuth. Audiobookshelf has its own internal user management — ForwardAuth guards the URL. Mobile app users authenticate directly to Audiobookshelf using their Audiobookshelf credentials (separate from Authelia).

!!! note "Mobile app access"
    The Audiobookshelf mobile apps connect directly to `https://audiobooks.yourdomain.com` and authenticate using Audiobookshelf's own username/password. This bypasses the Authelia ForwardAuth layer. This is expected — ForwardAuth only intercepts browser requests. Store the Audiobookshelf credentials in Vaultwarden.

## Step 1: NFS Volumes for Library

The audiobook and podcast files live on the NAS. Create static PVs pointing to the NAS NFS exports.

Create `homelab-manifests/apps/audiobookshelf/pv-library.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: abs-audiobooks
spec:
  capacity:
    storage: 500Gi
  accessModes: [ReadOnlyMany]
  nfs:
    server: <nas-ip>
    path: /volume1/media/audiobooks
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: abs-audiobooks
  namespace: audiobookshelf
spec:
  accessModes: [ReadOnlyMany]
  storageClassName: ""
  volumeName: abs-audiobooks
  resources:
    requests:
      storage: 500Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: abs-podcasts
spec:
  capacity:
    storage: 100Gi
  accessModes: [ReadWriteMany]
  nfs:
    server: <nas-ip>
    path: /volume1/media/podcasts
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: abs-podcasts
  namespace: audiobookshelf
spec:
  accessModes: [ReadWriteMany]
  storageClassName: ""
  volumeName: abs-podcasts
  resources:
    requests:
      storage: 100Gi
```

Podcasts is `ReadWriteMany` because Audiobookshelf downloads new episodes directly to the podcast directory.

## Step 2: Deploy Audiobookshelf

Create `homelab-manifests/apps/audiobookshelf/deployment.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: audiobookshelf
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: audiobookshelf
  namespace: audiobookshelf
spec:
  replicas: 1
  selector:
    matchLabels:
      app: audiobookshelf
  template:
    metadata:
      labels:
        app: audiobookshelf
    spec:
      containers:
        - name: audiobookshelf
          image: ghcr.io/advplyr/audiobookshelf:latest
          ports:
            - containerPort: 80
          env:
            - name: TZ
              value: Europe/London
          volumeMounts:
            - name: config
              mountPath: /config
            - name: metadata
              mountPath: /metadata
            - name: audiobooks
              mountPath: /audiobooks
            - name: podcasts
              mountPath: /podcasts
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: abs-config
        - name: metadata
          persistentVolumeClaim:
            claimName: abs-metadata
        - name: audiobooks
          persistentVolumeClaim:
            claimName: abs-audiobooks
        - name: podcasts
          persistentVolumeClaim:
            claimName: abs-podcasts
```

Create `homelab-manifests/apps/audiobookshelf/pvcs.yaml` for config and metadata:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: abs-config
  namespace: audiobookshelf
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: abs-metadata
  namespace: audiobookshelf
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 5Gi
```

Create `homelab-manifests/apps/audiobookshelf/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: audiobookshelf
  namespace: audiobookshelf
spec:
  selector:
    app: audiobookshelf
  ports:
    - port: 80
      targetPort: 80
```

## Step 3: IngressRoute

Create `homelab-manifests/apps/audiobookshelf/ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: audiobookshelf
  namespace: audiobookshelf
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`audiobooks.yourdomain.com`)
      kind: Rule
      middlewares:
        - name: authelia@kubernetescrd
          namespace: authelia
      services:
        - name: audiobookshelf
          port: 80
  tls:
    certResolver: cloudflare
    domains:
      - main: audiobooks.yourdomain.com
```

Commit all manifests to `homelab-manifests/apps/audiobookshelf/` and let ArgoCD sync.

## Step 4: Initial Setup

Open `https://audiobooks.yourdomain.com` (you will be prompted to log in with Authelia first). Create the admin user in the Audiobookshelf setup wizard. Save the credentials to Vaultwarden.

Add libraries:
1. **Add Library → Audiobooks** and set the path to `/audiobooks`.
2. **Add Library → Podcasts** and set the path to `/podcasts`.

Trigger a full library scan. Audiobookshelf will match audiobooks to metadata automatically.

## Step 5: Mobile App

Install the Audiobookshelf app on Android or iOS. Configure it with:
- **Server URL**: `https://audiobooks.yourdomain.com`
- **Username / Password**: the Audiobookshelf credentials (not Authelia)

Progress, bookmarks, and playlists sync automatically between devices.

## Verification

- [ ] Audiobookshelf pod Running:

    ```bash
    kubectl get pods -n audiobookshelf
    ```

- [ ] `https://audiobooks.yourdomain.com` redirects to Authelia for browser access.
- [ ] Library scan completes and audiobooks appear in the UI.
- [ ] Mobile app connects and shows the library.
- [ ] Credentials saved to Vaultwarden.
