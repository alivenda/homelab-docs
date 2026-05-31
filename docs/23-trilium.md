# Runbook 23: TriliumNext Notes

Self-hosted hierarchical note-taking with rich text, code blocks, relations, and scripting.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 20–30 minutes |
| **Runs On** | k3s (cluster) |
| **Depends On** | Runbook 5 (k3s), Runbook 6 (Traefik), Runbook 18 (Authelia — ForwardAuth) |

TriliumNext Notes ([github.com/TriliumNext/Notes](https://github.com/TriliumNext/Notes)) is an open-source fork of Trilium maintained by the community. It supports nested notes, code blocks, relations between notes, and a built-in scripting engine. ARM64 ✅ (`triliumnext/notes` publishes AMD64, ARMv7, and ARM64 builds). See the [installation documentation](https://triliumnext.github.io/Docs/Wiki/server-installation.html) for full reference.

**Auth mode:** Authelia ForwardAuth (TriliumNext has no OIDC integration — its own login is bypassed in single-user server mode). The server is protected at the Traefik layer by Authelia.

!!! note "Pin the image tag"
    The docs warn against running `latest` as it may auto-upgrade and disrupt desktop client sync. Pin to a specific version tag (e.g. `triliumnext/notes:v0.90.3`) and update deliberately.

## Step 1: Deploy TriliumNext

Create `homelab-manifests/apps/trilium/deployment.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: trilium
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trilium
  namespace: trilium
spec:
  replicas: 1
  selector:
    matchLabels:
      app: trilium
  template:
    metadata:
      labels:
        app: trilium
    spec:
      containers:
        - name: trilium
          image: triliumnext/notes:latest
          ports:
            - containerPort: 8080
          env:
            - name: USER_UID
              value: "1000"
            - name: USER_GID
              value: "1000"
            - name: TZ
              value: Europe/London
          volumeMounts:
            - name: data
              mountPath: /home/node/trilium-data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: trilium-data
```

Create `homelab-manifests/apps/trilium/pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: trilium-data
  namespace: trilium
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 5Gi
```

Create `homelab-manifests/apps/trilium/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: trilium
  namespace: trilium
spec:
  selector:
    app: trilium
  ports:
    - port: 8080
      targetPort: 8080
```

## Step 2: IngressRoute

Create `homelab-manifests/apps/trilium/ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: trilium
  namespace: trilium
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`notes.yourdomain.com`)
      kind: Rule
      middlewares:
        - name: authelia@kubernetescrd
          namespace: authelia
      services:
        - name: trilium
          port: 8080
  tls:
    certResolver: cloudflare
    domains:
      - main: notes.yourdomain.com
```

Commit all manifests to `homelab-manifests/apps/trilium/` and let ArgoCD sync.

## Step 3: First Login

Open `https://notes.yourdomain.com`. On first access, TriliumNext prompts you to set an **initial password**. Set a strong password and save it to Vaultwarden — this is the TriliumNext internal password used to encrypt the note database.

After setting the password, you will land in the TriliumNext interface. Authelia guards the URL; TriliumNext's own login prompt only appears once (on initial setup) and then trusts the active session.

## Step 4: Desktop Client Sync (Optional)

TriliumNext ships a desktop application (Electron) that syncs with the server instance. To connect it:

1. Download the TriliumNext desktop app from [GitHub Releases](https://github.com/TriliumNext/Notes/releases) — match the version pinned in your server deployment.
2. Open the desktop app and choose **Connect to server**.
3. Enter `https://notes.yourdomain.com` and the TriliumNext password set in Step 3.

The desktop app uses its own encrypted sync protocol — Authelia is not involved in the sync connection.

## Verification

- [ ] Trilium pod Running:

    ```bash
    kubectl get pods -n trilium
    ```

- [ ] `https://notes.yourdomain.com` redirects to Authelia for unauthenticated requests.
- [ ] Initial password setup completes and the note editor loads.
- [ ] A test note can be created and saved.
- [ ] TriliumNext password saved to Vaultwarden.
