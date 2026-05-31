# Runbook 29: Kavita

Self-hosted e-book, manga, and comic library with OPDS and OIDC support.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 30 minutes |
| **Runs On** | k3s (cluster) — library on NAS via NFS |
| **Depends On** | Runbook 5 (k3s), Runbook 6 (Traefik), Runbook 18 (Authelia OIDC) |

Kavita ([kavitareader.com](https://www.kavitareader.com)) is a fast e-book, manga, and comic server with native reading progress sync, OPDS feed support, and a polished web reader. ARM64 support — verify with `docker manifest inspect jvmilazz0/kavita:latest` before deploying. See the [Kavita documentation](https://wiki.kavitareader.com) for full reference.

**Auth mode:** OIDC via the Kavita admin UI (not environment variables). Kavita configures OIDC through its Settings panel rather than at deploy time.

## Step 1: NFS Volume for Library

Create `homelab-manifests/apps/kavita/pv-library.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: kavita-books
spec:
  capacity:
    storage: 500Gi
  accessModes: [ReadOnlyMany]
  nfs:
    server: <nas-ip>
    path: /volume1/media/books
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kavita-books
  namespace: kavita
spec:
  accessModes: [ReadOnlyMany]
  storageClassName: ""
  volumeName: kavita-books
  resources:
    requests:
      storage: 500Gi
```

## Step 2: Deploy Kavita

Create `homelab-manifests/apps/kavita/deployment.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kavita
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kavita
  namespace: kavita
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kavita
  template:
    metadata:
      labels:
        app: kavita
    spec:
      containers:
        - name: kavita
          image: jvmilazz0/kavita:latest
          ports:
            - containerPort: 5000
          env:
            - name: TZ
              value: Europe/London
          volumeMounts:
            - name: config
              mountPath: /kavita/config
            - name: books
              mountPath: /books
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: kavita-config
        - name: books
          persistentVolumeClaim:
            claimName: kavita-books
```

Create `homelab-manifests/apps/kavita/pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kavita-config
  namespace: kavita
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 2Gi
```

Create `homelab-manifests/apps/kavita/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kavita
  namespace: kavita
spec:
  selector:
    app: kavita
  ports:
    - port: 5000
      targetPort: 5000
```

## Step 3: IngressRoute

Create `homelab-manifests/apps/kavita/ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: kavita
  namespace: kavita
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`books.yourdomain.com`)
      kind: Rule
      services:
        - name: kavita
          port: 5000
  tls:
    certResolver: cloudflare
    domains:
      - main: books.yourdomain.com
```

No Authelia middleware — Kavita's OIDC handles authentication once configured in Step 5.

Commit all manifests to `homelab-manifests/apps/kavita/` and let ArgoCD sync.

## Step 4: Initial Setup

Open `https://books.yourdomain.com`. The first-run wizard asks you to create an admin user. Set a strong password and save to Vaultwarden — this is used before OIDC is configured and as a fallback admin account.

After the wizard, go to **Admin Dashboard → Libraries → Add Library** and point it to `/books`.

## Step 5: Configure OIDC

Kavita's OIDC is configured entirely in the Admin UI.

**In Authelia** — add the Kavita client in `values.yaml`:

```yaml
- client_id: kavita
  client_name: Kavita
  client_secret:
    value: <HASHED_SECRET>
  redirect_uris:
    - https://books.yourdomain.com/signin-oidc
  scopes: [openid, profile, email]
  token_endpoint_auth_method: client_secret_basic
  grant_types: [authorization_code]
  response_types: [code]
```

Commit and upgrade Authelia before proceeding.

**In Kavita Admin Dashboard → Settings → Authentication:**

1. Enable **OpenID Connect**.
2. Set **Authority** to `https://auth.yourdomain.com` (Authelia issuer URL).
3. Set **Client ID** to `kavita`.
4. Set **Client Secret** to the plaintext OIDC secret from Vaultwarden.
5. Save and restart Kavita (restart the pod):

    ```bash
    kubectl rollout restart -n kavita deployment/kavita
    ```

!!! warning "Restart required"
    Changes to Authority, Client ID, and Secret in the Kavita admin panel take effect only after a pod restart.

After restart, the Kavita login page will show a **Sign in with OIDC** button.

## Verification

- [ ] Kavita pod Running:

    ```bash
    kubectl get pods -n kavita
    ```

- [ ] `https://books.yourdomain.com` loads the login page.
- [ ] **Sign in with OIDC** button appears and redirects to Authelia.
- [ ] Library scan completes and books appear in the web reader.
- [ ] Admin credentials and OIDC client secret saved to Vaultwarden.
