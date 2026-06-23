# Reactive Resume

!!! info "Status — Planned (not yet deployed)"
    Documented — not yet deployed in this build.

Self-hosted resume builder with OIDC login, PDF export, and a polished editor.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 45–60 minutes |
| **Runs On** | k3s (cluster) |
| **Depends On** | Kubernetes (k3s), Traefik, Authelia (OIDC) |

Reactive Resume ([rxresu.me](https://rxresu.me)) is a feature-complete resume builder with real-time editing, multiple templates, and PDF export. ARM64 ✅ — the image is based on `node:24-slim` (multi-arch), and `FLAG_DISABLE_IMAGE_PROCESSING=true` reduces load on ARM hardware. See the [self-hosting docs](https://docs.rxresu.me/overview/self-hosting) for full reference.

**Auth mode:** OIDC client (Authelia). Reactive Resume has its own user accounts and OIDC integration via standard env vars. Do **not** use ForwardAuth.

Reactive Resume requires PostgreSQL and Redis in addition to the app container.

## Step 1: SealedSecrets

```bash
kubectl create namespace reactive-resume

kubectl create secret generic rxresume-secrets \
  --namespace reactive-resume \
  --from-literal=AUTH_SECRET=$(openssl rand -hex 32) \
  --from-literal=DATABASE_PASSWORD=$(openssl rand -base64 32) \
  --from-literal=OAUTH_CLIENT_SECRET=<oidc_client_secret_plaintext> \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > rxresume-secrets-sealed.yaml
```

Save all plaintext values to Vaultwarden.

## Step 2: Register the OIDC Client in Authelia

In `homelab-manifests/apps/authelia/values.yaml`, add under `configMap.identity_providers.oidc.clients`:

```yaml
- client_id: reactive-resume
  client_name: Reactive Resume
  client_secret:
    value: <HASHED_SECRET>   # authelia crypto hash generate pbkdf2 --variant sha512
  redirect_uris:
    - https://resume.yourdomain.com/api/auth/callback/custom-oidc
  scopes: [openid, profile, email]
  token_endpoint_auth_method: client_secret_basic
  grant_types: [authorization_code]
  response_types: [code]
```

Commit and upgrade Authelia before deploying Reactive Resume.

## Step 3: PostgreSQL and Redis

Create `homelab-manifests/apps/reactive-resume/postgres.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rxresume-postgres
  namespace: reactive-resume
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rxresume-postgres
  template:
    metadata:
      labels:
        app: rxresume-postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          env:
            - name: POSTGRES_DB
              value: resume
            - name: POSTGRES_USER
              value: rxresume
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: rxresume-secrets
                  key: DATABASE_PASSWORD
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: rxresume-postgres
---
apiVersion: v1
kind: Service
metadata:
  name: rxresume-postgres
  namespace: reactive-resume
spec:
  selector:
    app: rxresume-postgres
  ports:
    - port: 5432
      targetPort: 5432
```

Create `homelab-manifests/apps/reactive-resume/redis.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rxresume-redis
  namespace: reactive-resume
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rxresume-redis
  template:
    metadata:
      labels:
        app: rxresume-redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: rxresume-redis
  namespace: reactive-resume
spec:
  selector:
    app: rxresume-redis
  ports:
    - port: 6379
      targetPort: 6379
```

Create `homelab-manifests/apps/reactive-resume/pvcs.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rxresume-postgres
  namespace: reactive-resume
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rxresume-data
  namespace: reactive-resume
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 2Gi
```

## Step 4: Deploy Reactive Resume

Create `homelab-manifests/apps/reactive-resume/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reactive-resume
  namespace: reactive-resume
spec:
  replicas: 1
  selector:
    matchLabels:
      app: reactive-resume
  template:
    metadata:
      labels:
        app: reactive-resume
    spec:
      initContainers:
        - name: wait-for-db
          image: busybox:latest
          command: ['sh', '-c', 'until nc -z rxresume-postgres 5432; do sleep 2; done']
      containers:
        - name: reactive-resume
          image: ghcr.io/amruthpillai/reactive-resume:latest
          ports:
            - containerPort: 3000
          env:
            - name: APP_URL
              value: https://resume.yourdomain.com
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: rxresume-secrets
                  key: DATABASE_PASSWORD
            - name: DATABASE_URL
              value: postgresql://rxresume:$(DATABASE_PASSWORD)@rxresume-postgres:5432/resume
            - name: REDIS_URL
              value: redis://rxresume-redis:6379
            - name: FLAG_DISABLE_SIGNUPS
              value: "true"
            - name: FLAG_DISABLE_IMAGE_PROCESSING
              value: "true"
            - name: OAUTH_PROVIDER_NAME
              value: Authelia
            - name: OAUTH_DISCOVERY_URL
              value: https://auth.yourdomain.com/.well-known/openid-configuration
            - name: OAUTH_CLIENT_ID
              value: reactive-resume
            - name: OAUTH_SCOPES
              value: "openid profile email"
          envFrom:
            - secretRef:
                name: rxresume-secrets
          volumeMounts:
            - name: data
              mountPath: /app/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: rxresume-data
```

!!! note "DATABASE_URL interpolation"
    Kubernetes `$(VAR)` expansion only works against earlier entries in the same `env` list — not against `envFrom`. `DATABASE_PASSWORD` is declared as an explicit `env` entry with `valueFrom.secretKeyRef` **before** `DATABASE_URL` so the interpolation resolves at pod start.

Create `homelab-manifests/apps/reactive-resume/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: reactive-resume
  namespace: reactive-resume
spec:
  selector:
    app: reactive-resume
  ports:
    - port: 3000
      targetPort: 3000
```

## Step 5: HTTPRoute

Create `homelab-manifests/apps/reactive-resume/httproute.yaml` (TLS is handled by the Gateway's wildcard cert — see [Deploying an App](apps-deploy-pattern.md)):

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reactive-resume
  namespace: reactive-resume
spec:
  parentRefs:
    - name: traefik
      namespace: traefik
      sectionName: websecure
  hostnames:
    - resume.yourdomain.com
  rules:
    - backendRefs:
        - name: reactive-resume
          port: 3000
```

Commit all manifests to `homelab-manifests/apps/reactive-resume/` and let ArgoCD sync.

## Step 6: First Login

Open `https://resume.yourdomain.com`. Click **Continue with Authelia**. Reactive Resume creates a local user on first OIDC login.

`FLAG_DISABLE_SIGNUPS=true` prevents any new accounts except via OIDC. `FLAG_DISABLE_IMAGE_PROCESSING=true` disables server-side image resizing — useful on ARM hardware; resume profile images still work but are not server-processed.

## Verification

- [ ] All three pods Running (postgres, redis, app):

    ```bash
    kubectl get pods -n reactive-resume
    ```

- [ ] `https://resume.yourdomain.com` loads the login page.
- [ ] OIDC login completes and the resume dashboard loads.
- [ ] A test resume can be created and exported as PDF.
