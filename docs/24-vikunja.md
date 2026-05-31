# Runbook 24: Vikunja

Self-hosted task management with OIDC login, projects, teams, and a Kanban view.

| | |
|---|---|
| **Difficulty** | Beginner–Intermediate |
| **Time Estimate** | 45–60 minutes |
| **Runs On** | k3s (cluster) |
| **Depends On** | Runbook 5 (k3s), Runbook 6 (Traefik), Runbook 18 (Authelia OIDC) |

Vikunja ([vikunja.io](https://vikunja.io)) is a feature-complete task manager with lists, Kanban, Gantt, and team sharing. Frontend and API ship as a single container. ARM64 ✅ (`vikunja/vikunja` publishes multiarch including ARM64). See the [official documentation](https://vikunja.io/docs/) for full reference.

**Auth mode:** OIDC client (Authelia). Vikunja has its own user system. Do **not** use ForwardAuth.

## Step 1: SealedSecret

```bash
kubectl create namespace vikunja

kubectl create secret generic vikunja-secrets \
  --namespace vikunja \
  --from-literal=VIKUNJA_AUTH_OPENID_PROVIDERS_AUTHELIA_CLIENTSECRET=<oidc_client_secret_plaintext> \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > vikunja-secrets-sealed.yaml
```

Save the plaintext client secret to Vaultwarden.

## Step 2: Register the OIDC Client in Authelia

In `homelab-manifests/apps/authelia/values.yaml`, add under `configMap.identity_providers.oidc.clients`:

```yaml
- client_id: vikunja
  client_name: Vikunja
  client_secret:
    value: <HASHED_SECRET>   # authelia crypto hash generate pbkdf2 --variant sha512
  redirect_uris:
    - https://tasks.yourdomain.com/auth/openid/authelia
  scopes: [openid, profile, email, groups]
  token_endpoint_auth_method: client_secret_basic
  grant_types: [authorization_code]
  response_types: [code]
```

!!! note "Redirect URI format"
    Vikunja's OIDC callback path is `/auth/openid/<provider-name>` where the provider name matches the key you set in `VIKUNJA_AUTH_OPENID_PROVIDERS_*_NAME`.

Commit the updated `values.yaml` and upgrade the Authelia Helm release before deploying Vikunja.

## Step 3: Deploy Vikunja

Create `homelab-manifests/apps/vikunja/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vikunja
  namespace: vikunja
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vikunja
  template:
    metadata:
      labels:
        app: vikunja
    spec:
      containers:
        - name: vikunja
          image: vikunja/vikunja:latest
          ports:
            - containerPort: 3456
          env:
            - name: VIKUNJA_SERVICE_PUBLICURL
              value: https://tasks.yourdomain.com
            - name: VIKUNJA_DATABASE_TYPE
              value: sqlite
            - name: VIKUNJA_AUTH_OPENID_ENABLED
              value: "true"
            - name: VIKUNJA_AUTH_OPENID_PROVIDERS_AUTHELIA_NAME
              value: authelia
            - name: VIKUNJA_AUTH_OPENID_PROVIDERS_AUTHELIA_AUTHURL
              value: https://auth.yourdomain.com
            - name: VIKUNJA_AUTH_OPENID_PROVIDERS_AUTHELIA_CLIENTID
              value: vikunja
          envFrom:
            - secretRef:
                name: vikunja-secrets
          volumeMounts:
            - name: files
              mountPath: /app/vikunja/files
            - name: db
              mountPath: /db
      volumes:
        - name: files
          persistentVolumeClaim:
            claimName: vikunja-files
        - name: db
          persistentVolumeClaim:
            claimName: vikunja-db
```

Create `homelab-manifests/apps/vikunja/pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vikunja-files
  namespace: vikunja
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 2Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vikunja-db
  namespace: vikunja
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 1Gi
```

Create `homelab-manifests/apps/vikunja/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: vikunja
  namespace: vikunja
spec:
  selector:
    app: vikunja
  ports:
    - port: 3456
      targetPort: 3456
```

## Step 4: IngressRoute

Create `homelab-manifests/apps/vikunja/ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: vikunja
  namespace: vikunja
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`tasks.yourdomain.com`)
      kind: Rule
      services:
        - name: vikunja
          port: 3456
  tls:
    certResolver: cloudflare
    domains:
      - main: tasks.yourdomain.com
```

No Authelia middleware here — Vikunja's OIDC integration handles authentication.

Commit all manifests to `homelab-manifests/apps/vikunja/` and let ArgoCD sync.

## Step 5: First Login

Open `https://tasks.yourdomain.com`. The login page shows a **Log in with authelia** button. Click it to authenticate via Authelia. Vikunja creates a local user record on first OIDC login.

!!! note "OIDC discovery"
    Vikunja uses OIDC discovery — it fetches `https://auth.yourdomain.com/.well-known/openid-configuration` on startup to resolve token and userinfo endpoints automatically. No need to specify them separately.

## Verification

- [ ] Vikunja pod Running:

    ```bash
    kubectl get pods -n vikunja
    ```

- [ ] `https://tasks.yourdomain.com` loads the login page.
- [ ] **Log in with authelia** button is present on the login page.
- [ ] OIDC login completes and Vikunja home screen loads.
- [ ] A test task can be created and completed.
