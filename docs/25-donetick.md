# Runbook 25: Donetick

Self-hosted chore and habit tracker with recurring schedules, assignees, and OIDC login.

| | |
|---|---|
| **Difficulty** | Beginner–Intermediate |
| **Time Estimate** | 30–45 minutes |
| **Runs On** | k3s (cluster) |
| **Depends On** | Runbook 5 (k3s), Runbook 6 (Traefik), Runbook 18 (Authelia OIDC) |

Donetick ([donetick.com](https://donetick.com)) is a household chore and recurring-task tracker. Unlike Vikunja (project/task management), Donetick focuses on recurring schedules, assignment to household members, and due-date tracking for repeating chores. ARM64 support — verify with `docker manifest inspect donetick/donetick:latest` before deploying on ARM64. See the [Donetick repository](https://github.com/donetick/donetick) for full reference.

**Auth mode:** OIDC client (Authelia). Donetick has its own user system. Do **not** use ForwardAuth.

## Step 1: SealedSecret

```bash
kubectl create namespace donetick

kubectl create secret generic donetick-secrets \
  --namespace donetick \
  --from-literal=JWT_SECRET=$(openssl rand -base64 32) \
  --from-literal=OAUTH2_CLIENT_SECRET=<oidc_client_secret_plaintext> \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > donetick-secrets-sealed.yaml
```

Save both values to Vaultwarden.

## Step 2: Register the OIDC Client in Authelia

In `homelab-manifests/apps/authelia/values.yaml`, add under `configMap.identity_providers.oidc.clients`:

```yaml
- client_id: donetick
  client_name: Donetick
  client_secret:
    value: <HASHED_SECRET>   # authelia crypto hash generate pbkdf2 --variant sha512
  redirect_uris:
    - https://chores.yourdomain.com/api/v1/auth/oauth2/callback
  scopes: [openid, profile, email, groups]
  token_endpoint_auth_method: client_secret_basic
  grant_types: [authorization_code]
  response_types: [code]
```

Commit the updated `values.yaml` and upgrade the Authelia Helm release before deploying Donetick.

## Step 3: ConfigMap

Donetick is configured via a `selfhosted.yaml` file mounted into the container. The OAuth2 section requires all endpoints to be specified explicitly — Donetick does not use OIDC discovery.

Create `homelab-manifests/apps/donetick/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: donetick-config
  namespace: donetick
data:
  selfhosted.yaml: |
    name: "selfhosted"
    is_done_tick_dot_com: false
    is_user_creation_disabled: false
    database:
      type: "sqlite"
      migration: true
    server:
      port: 2021
      read_timeout: 10s
      write_timeout: 10s
      rate_period: 60s
      rate_limit: 300
      cors_allow_origins:
        - "https://chores.yourdomain.com"
        - "capacitor://localhost"
        - "http://localhost"
      serve_frontend: true
      public_host: "https://chores.yourdomain.com"
    logging:
      level: "info"
      encoding: "json"
    scheduler_jobs:
      due_job: 30m
      overdue_job: 3h
      pre_due_job: 3h
    oauth2:
      client_id: donetick
      auth_url: https://auth.yourdomain.com/api/oidc/authorization
      token_url: https://auth.yourdomain.com/api/oidc/token
      user_info_url: https://auth.yourdomain.com/api/oidc/userinfo
      redirect_url: https://chores.yourdomain.com/api/v1/auth/oauth2/callback
      name: Authelia
    realtime:
      enabled: true
      sse_enabled: true
      heartbeat_interval: 60s
      connection_timeout: 120s
      max_connections: 1000
```

!!! note "OAuth2 endpoints"
    Donetick does **not** use OIDC discovery. You must specify `auth_url`, `token_url`, and `user_info_url` separately. The `client_secret` is injected from the SealedSecret as an environment variable — it must **not** appear in the ConfigMap.

## Step 4: Deploy Donetick

Create `homelab-manifests/apps/donetick/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: donetick
  namespace: donetick
spec:
  replicas: 1
  selector:
    matchLabels:
      app: donetick
  template:
    metadata:
      labels:
        app: donetick
    spec:
      containers:
        - name: donetick
          image: donetick/donetick:latest
          ports:
            - containerPort: 2021
          env:
            - name: DT_ENV
              value: selfhosted
            - name: DT_SQLITE_PATH
              value: /donetick-data/donetick.db
          envFrom:
            - secretRef:
                name: donetick-secrets
          volumeMounts:
            - name: data
              mountPath: /donetick-data
            - name: config
              mountPath: /config
          livenessProbe:
            httpGet:
              path: /api/v1/health
              port: 2021
            initialDelaySeconds: 5
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: donetick-data
        - name: config
          configMap:
            name: donetick-config
```

Create `homelab-manifests/apps/donetick/pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: donetick-data
  namespace: donetick
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 1Gi
```

Create `homelab-manifests/apps/donetick/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: donetick
  namespace: donetick
spec:
  selector:
    app: donetick
  ports:
    - port: 2021
      targetPort: 2021
```

## Step 5: IngressRoute

Create `homelab-manifests/apps/donetick/ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: donetick
  namespace: donetick
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`chores.yourdomain.com`)
      kind: Rule
      services:
        - name: donetick
          port: 2021
  tls:
    certResolver: cloudflare
    domains:
      - main: chores.yourdomain.com
```

Commit all manifests to `homelab-manifests/apps/donetick/` and let ArgoCD sync.

## Step 6: First Login

Open `https://chores.yourdomain.com`. Click **Login with Authelia** to authenticate. Donetick creates a local user on first OIDC login.

To set up a household: go to **Circles** and create or invite members. With `single_circle_instance: true` in the config, all new OIDC users are automatically added to a shared circle instead of creating personal ones — useful if this is a family/shared install.

## Verification

- [ ] Donetick pod Running:

    ```bash
    kubectl get pods -n donetick
    ```

- [ ] Health endpoint responds:

    ```bash
    kubectl port-forward -n donetick svc/donetick 2021:2021
    curl http://localhost:2021/api/v1/health
    ```

- [ ] `https://chores.yourdomain.com` loads the login page.
- [ ] OIDC login via Authelia completes successfully.
- [ ] A recurring chore can be created and marked complete.
