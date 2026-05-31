# Runbook 27: Arr Stack

Media automation: Sonarr, Radarr, Lidarr, Prowlarr, Seerr, and qBittorrent.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 2–3 hours |
| **Runs On** | k3s (cluster) — see NAS migration note |
| **Depends On** | Runbook 5 (k3s), Runbook 6 (Traefik), Runbook 18 (Authelia — ForwardAuth) |

The arr stack automates downloading and organising TV shows, films, and music. Prowlarr indexes torrent trackers and Usenet indexers in one place; Sonarr/Radarr/Lidarr handle search and post-processing; qBittorrent is the download client; Seerr provides a Plex-integrated request interface. ARM64 ✅ (all LinuxServer.io images ship ARM64). See the [Servarr documentation](https://wiki.servarr.com) for full reference.

**Auth mode:** Authelia ForwardAuth protects all arr services — they have no meaningful OIDC support and their API keys are used by integrations, not users.

## Hardlink note — read before deploying

The Servarr wiki's strongest recommendation is a **unified `/data` layout** where download client and media library share the same filesystem path. This allows atomic hardlink renames instead of slow copy+delete operations.

**Current state (8 GB NAS RAM):** arr apps run on the cluster via a single NFS mount. Hardlinks require that source and destination live on the same filesystem — since the cluster mounts `/data` over NFS, moves from `/data/downloads` to `/data/media` cross a network boundary and are done as copy+delete. This works but doubles NAS I/O during post-processing.

**After 16 GB NAS RAM upgrade:** move the entire arr stack to NAS Docker Compose with both the download client and Plex sharing the same local filesystem. This enables native hardlinks and eliminates the double-copy. Each app's runbook section notes when this matters.

## Step 1: Shared Infrastructure

All arr apps use the same NFS PVC structure. Create the namespace and the shared volumes first.

Create `homelab-manifests/apps/arr/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: arr
```

Create `homelab-manifests/apps/arr/pvcs.yaml`. Each app gets its own config PVC; media paths are shared via NFS:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sonarr-config
  namespace: arr
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
  name: radarr-config
  namespace: arr
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
  name: lidarr-config
  namespace: arr
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
  name: prowlarr-config
  namespace: arr
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
  name: qbittorrent-config
  namespace: arr
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 1Gi
```

### NFS volume for data

All arr apps and the download client mount a **single NFS export** as `/data`. Downloads land under `/data/downloads/` and the media library lives under `/data/media/`. Sharing one filesystem root is what makes hardlinks possible — on the cluster the files are still copied (NFS boundary), but when you migrate to NAS Docker after the RAM upgrade, both qBittorrent and Plex will use the same local filesystem and the hardlinks work natively.

Create the NAS directory layout first (SSH into the NAS):

```bash
mkdir -p /volume1/data/downloads/{movies,tv,music}
mkdir -p /volume1/data/media/{movies,tv,music}
```

Create `homelab-manifests/apps/arr/pv-data.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: arr-data
spec:
  capacity:
    storage: 2Ti
  accessModes: [ReadWriteMany]
  nfs:
    server: <nas-ip>
    path: /volume1/data
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: arr-data
  namespace: arr
spec:
  accessModes: [ReadWriteMany]
  storageClassName: ""
  volumeName: arr-data
  resources:
    requests:
      storage: 2Ti
```

## Step 2: qBittorrent

Create `homelab-manifests/apps/arr/qbittorrent.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: qbittorrent
  namespace: arr
spec:
  replicas: 1
  selector:
    matchLabels:
      app: qbittorrent
  template:
    metadata:
      labels:
        app: qbittorrent
    spec:
      containers:
        - name: qbittorrent
          image: lscr.io/linuxserver/qbittorrent:latest
          ports:
            - containerPort: 8080
            - containerPort: 6881
            - containerPort: 6881
              protocol: UDP
          env:
            - name: PUID
              value: "1000"
            - name: PGID
              value: "1000"
            - name: TZ
              value: Europe/London
            - name: WEBUI_PORT
              value: "8080"
          volumeMounts:
            - name: config
              mountPath: /config
            - name: data
              mountPath: /data
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: qbittorrent-config
        - name: data
          persistentVolumeClaim:
            claimName: arr-data
---
apiVersion: v1
kind: Service
metadata:
  name: qbittorrent
  namespace: arr
spec:
  selector:
    app: qbittorrent
  ports:
    - name: webui
      port: 8080
      targetPort: 8080
```

## Step 3: Prowlarr

Create `homelab-manifests/apps/arr/prowlarr.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prowlarr
  namespace: arr
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prowlarr
  template:
    metadata:
      labels:
        app: prowlarr
    spec:
      containers:
        - name: prowlarr
          image: lscr.io/linuxserver/prowlarr:latest
          ports:
            - containerPort: 9696
          env:
            - name: PUID
              value: "1000"
            - name: PGID
              value: "1000"
            - name: TZ
              value: Europe/London
          volumeMounts:
            - name: config
              mountPath: /config
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: prowlarr-config
---
apiVersion: v1
kind: Service
metadata:
  name: prowlarr
  namespace: arr
spec:
  selector:
    app: prowlarr
  ports:
    - port: 9696
      targetPort: 9696
```

## Step 4: Sonarr, Radarr, Lidarr

All three follow the same pattern — the only differences are image name, port, and config PVC. Create `homelab-manifests/apps/arr/sonarr.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sonarr
  namespace: arr
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sonarr
  template:
    metadata:
      labels:
        app: sonarr
    spec:
      containers:
        - name: sonarr
          image: lscr.io/linuxserver/sonarr:latest
          ports:
            - containerPort: 8989
          env:
            - name: PUID
              value: "1000"
            - name: PGID
              value: "1000"
            - name: TZ
              value: Europe/London
          volumeMounts:
            - name: config
              mountPath: /config
            - name: data
              mountPath: /data
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: sonarr-config
        - name: data
          persistentVolumeClaim:
            claimName: arr-data
---
apiVersion: v1
kind: Service
metadata:
  name: sonarr
  namespace: arr
spec:
  selector:
    app: sonarr
  ports:
    - port: 8989
      targetPort: 8989
```

Duplicate the above for **Radarr** (port 7878, image `linuxserver/radarr`, config PVC `radarr-config`) and **Lidarr** (port 8686, image `linuxserver/lidarr`, config PVC `lidarr-config`). All three share the same `arr-data` PVC.

## Step 5: Seerr (Request Interface)

[Seerr](https://github.com/seerr-team/seerr) is the actively maintained successor to Jellyseerr and Overseerr. It integrates with Plex for library awareness and lets users request new content. Create `homelab-manifests/apps/arr/seerr.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: seerr
  namespace: arr
spec:
  replicas: 1
  selector:
    matchLabels:
      app: seerr
  template:
    metadata:
      labels:
        app: seerr
    spec:
      containers:
        - name: seerr
          image: seerr/seerr:latest
          ports:
            - containerPort: 5055
          volumeMounts:
            - name: config
              mountPath: /app/config
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: seerr-config
---
apiVersion: v1
kind: Service
metadata:
  name: seerr
  namespace: arr
spec:
  selector:
    app: seerr
  ports:
    - port: 5055
      targetPort: 5055
```

Add a `seerr-config` entry to `pvcs.yaml` (1 Gi, nfs-storage).

## Step 6: IngressRoutes

Create `homelab-manifests/apps/arr/ingressroutes.yaml`. All arr UIs are protected by Authelia ForwardAuth:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: sonarr
  namespace: arr
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`sonarr.yourdomain.com`)
      kind: Rule
      middlewares:
        - name: authelia@kubernetescrd
          namespace: authelia
      services:
        - name: sonarr
          port: 8989
  tls:
    certResolver: cloudflare
    domains:
      - main: sonarr.yourdomain.com
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: radarr
  namespace: arr
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`radarr.yourdomain.com`)
      kind: Rule
      middlewares:
        - name: authelia@kubernetescrd
          namespace: authelia
      services:
        - name: radarr
          port: 7878
  tls:
    certResolver: cloudflare
    domains:
      - main: radarr.yourdomain.com
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: lidarr
  namespace: arr
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`lidarr.yourdomain.com`)
      kind: Rule
      middlewares:
        - name: authelia@kubernetescrd
          namespace: authelia
      services:
        - name: lidarr
          port: 8686
  tls:
    certResolver: cloudflare
    domains:
      - main: lidarr.yourdomain.com
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: prowlarr
  namespace: arr
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`prowlarr.yourdomain.com`)
      kind: Rule
      middlewares:
        - name: authelia@kubernetescrd
          namespace: authelia
      services:
        - name: prowlarr
          port: 9696
  tls:
    certResolver: cloudflare
    domains:
      - main: prowlarr.yourdomain.com
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: qbittorrent
  namespace: arr
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`qbt.yourdomain.com`)
      kind: Rule
      middlewares:
        - name: authelia@kubernetescrd
          namespace: authelia
      services:
        - name: qbittorrent
          port: 8080
  tls:
    certResolver: cloudflare
    domains:
      - main: qbt.yourdomain.com
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: seerr
  namespace: arr
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`request.yourdomain.com`)
      kind: Rule
      services:
        - name: seerr
          port: 5055
  tls:
    certResolver: cloudflare
    domains:
      - main: request.yourdomain.com
```

!!! note "Seerr has no Authelia middleware"
    Seerr has its own login system and is meant to be shared with other users (Plex-authenticated). Leave it without Authelia ForwardAuth so Plex users can access it directly.

Commit all manifests to `homelab-manifests/apps/arr/` and let ArgoCD sync.

## Step 7: Initial Configuration

### qBittorrent

Open `https://qbt.yourdomain.com`. Default credentials are `admin` / `adminadmin`. Change the password immediately in Settings → Web UI. Set download path to `/data/downloads`.

### Prowlarr

Open `https://prowlarr.yourdomain.com`. Go to **Settings → Indexers → Add Indexer** and configure your torrent trackers and Usenet indexers. Then go to **Settings → Apps** and add Sonarr, Radarr, and Lidarr using their cluster-internal URLs:

| App | Internal URL |
|-----|-------------|
| Sonarr | `http://sonarr.arr.svc.cluster.local:8989` |
| Radarr | `http://radarr.arr.svc.cluster.local:7878` |
| Lidarr | `http://lidarr.arr.svc.cluster.local:8686` |

### Sonarr / Radarr / Lidarr

In each app:
1. **Settings → Media Management** — set your media root folder to `/data/media/tv` (Sonarr), `/data/media/movies` (Radarr), `/data/media/music` (Lidarr).
2. **Settings → Download Clients → Add** — add qBittorrent at `http://qbittorrent.arr.svc.cluster.local:8080`.
3. Prowlarr will automatically push indexer configs once connected.

### Seerr

Open `https://request.yourdomain.com` and follow the setup wizard. Connect your Plex server and the arr apps using their cluster-internal URLs. Plex users can now request content from the Seerr interface.

## NAS migration (after 16 GB RAM upgrade)

Once the NAS has 16 GB RAM, migrate the arr stack off the cluster for native hardlinks:

1. Stop all arr deployments in k3s.
2. Copy config directories from the NFS PVCs to NAS local storage.
3. Create a `docker-compose.yml` on the NAS following the Servarr unified `/data` layout.
4. Delete the k3s deployments and Ingresses (keep the PVCs until the migration is verified).
5. Expose the NAS arr UIs via Traefik reverse proxy entries (Traefik is already running on the cluster — add ServersTransport + IngressRoutes pointing to the NAS IP).

## Verification

- [ ] All arr pods Running:

    ```bash
    kubectl get pods -n arr
    ```

- [ ] All web UIs accessible and protected by Authelia.
- [ ] Prowlarr shows connected status in Settings → Apps for Sonarr, Radarr, and Lidarr.
- [ ] A test download completes and the file appears in the correct media folder.
- [ ] Seerr setup wizard completes; Plex library appears in Seerr.
- [ ] All app API keys saved to Vaultwarden.
