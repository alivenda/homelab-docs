# Runbook 21: FreshRSS

Self-hosted RSS aggregator with multi-user support and mobile sync APIs.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 30–45 minutes |
| **Runs On** | k3s (cluster) |
| **Depends On** | Runbook 5 (k3s), Runbook 6 (Traefik), Runbook 18 (Authelia OIDC) |

FreshRSS ([freshrss.org](https://freshrss.org)) is a fast, low-memory RSS/Atom aggregator. It exposes a Google Reader and Fever API so clients like Reeder, NetNewsWire, and Readrops can sync feeds and read state. The official `freshrss/freshrss` image ships Apache with `mod_auth_openidc` bundled — OIDC is handled natively without a separate proxy. ARM64 ✅ (official image is multi-arch). See the [FreshRSS documentation](https://freshrss.github.io/FreshRSS/) for full reference.

**Auth mode:** OIDC client (Authelia) via `mod_auth_openidc`. FreshRSS has its own user system. Mobile sync API clients authenticate with a separate FreshRSS API password — unaffected by OIDC.

## Step 1: SealedSecret

OIDC requires a client secret and an internal crypto key (used by `mod_auth_openidc` for session encryption):

```bash
kubectl create namespace freshrss

kubectl create secret generic freshrss-secrets \
  --namespace freshrss \
  --from-literal=OIDC_CLIENT_SECRET=<oidc_client_secret_plaintext> \
  --from-literal=OIDC_CLIENT_CRYPTO_KEY=$(openssl rand -hex 32) \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > freshrss-secrets-sealed.yaml
```

Save both values to Vaultwarden.

## Step 2: Register the OIDC Client in Authelia

In `homelab-manifests/apps/authelia/values.yaml`, add under `configMap.identity_providers.oidc.clients`:

```yaml
- client_id: freshrss
  client_name: FreshRSS
  client_secret:
    value: <HASHED_SECRET>   # authelia crypto hash generate pbkdf2 --variant sha512
  redirect_uris:
    - https://rss.yourdomain.com/i/oidc/
  scopes: [openid, profile, email]
  token_endpoint_auth_method: client_secret_basic
  grant_types: [authorization_code]
  response_types: [code]
```

!!! warning "Trailing slash"
    The FreshRSS OIDC callback URL requires a trailing slash: `/i/oidc/`.

Commit and upgrade Authelia before deploying FreshRSS.

## Step 3: Deploy FreshRSS

Create `homelab-manifests/apps/freshrss/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: freshrss
  namespace: freshrss
spec:
  replicas: 1
  selector:
    matchLabels:
      app: freshrss
  template:
    metadata:
      labels:
        app: freshrss
    spec:
      containers:
        - name: freshrss
          image: freshrss/freshrss:latest
          ports:
            - containerPort: 80
          env:
            - name: TZ
              value: Europe/London
            - name: CRON_MIN
              value: "2,32"
            - name: BASE_URL
              value: https://rss.yourdomain.com
            - name: OIDC_ENABLED
              value: "1"
            - name: OIDC_PROVIDER_METADATA_URL
              value: https://auth.yourdomain.com/.well-known/openid-configuration
            - name: OIDC_CLIENT_ID
              value: freshrss
            - name: OIDC_REMOTE_USER_CLAIM
              value: preferred_username
            - name: OIDC_SCOPES
              value: openid profile email
            - name: OIDC_X_FORWARDED_HEADERS
              value: "X-Forwarded-Host X-Forwarded-Port X-Forwarded-Proto"
          envFrom:
            - secretRef:
                name: freshrss-secrets
          volumeMounts:
            - name: data
              mountPath: /var/www/FreshRSS/data
            - name: extensions
              mountPath: /var/www/FreshRSS/extensions
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: freshrss-data
        - name: extensions
          persistentVolumeClaim:
            claimName: freshrss-extensions
```

Create `homelab-manifests/apps/freshrss/pvcs.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: freshrss-data
  namespace: freshrss
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 2Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: freshrss-extensions
  namespace: freshrss
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 512Mi
```

Create `homelab-manifests/apps/freshrss/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: freshrss
  namespace: freshrss
spec:
  selector:
    app: freshrss
  ports:
    - port: 80
      targetPort: 80
```

## Step 4: IngressRoute

Create `homelab-manifests/apps/freshrss/ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: freshrss
  namespace: freshrss
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`rss.yourdomain.com`)
      kind: Rule
      services:
        - name: freshrss
          port: 80
  tls:
    certResolver: cloudflare
    domains:
      - main: rss.yourdomain.com
```

No Authelia middleware — FreshRSS handles OIDC authentication natively via `mod_auth_openidc`.

Commit all manifests to `homelab-manifests/apps/freshrss/` and let ArgoCD sync.

## Step 5: Initial Setup

Open `https://rss.yourdomain.com`. The first-run wizard will prompt for:

1. **Default user** — enter your lldap username. This must match the `preferred_username` claim Authelia returns.
2. **Authentication method** — choose **HTTP authentication (mod_auth_openidc / OIDC)**. This is the step that activates OIDC login.

!!! warning "Authentication method must be set correctly during setup"
    If you choose the wrong auth method here, OIDC won't work. You can reset it by editing `/var/www/FreshRSS/data/config.php` and setting `'auth_type' => 'http_auth'`.

After setup, visiting `https://rss.yourdomain.com` redirects to Authelia login and back.

## Step 6: API Access for Mobile Clients

Mobile RSS clients authenticate directly against FreshRSS using its own API password — they bypass OIDC entirely.

In the FreshRSS web UI:

1. Go to **User profile → API management**.
2. Set an **API password** (distinct from any Authelia credentials).
3. Configure your mobile client:
   - **Server URL**: `https://rss.yourdomain.com`
   - **Username**: your FreshRSS username
   - **Password**: the API password

Save the API password to Vaultwarden.

## Verification

- [ ] FreshRSS pod Running:

    ```bash
    kubectl get pods -n freshrss
    ```

- [ ] `https://rss.yourdomain.com` redirects to Authelia login.
- [ ] After login, FreshRSS loads without prompting for its own password.
- [ ] A test feed (e.g. `https://blog.cloudflare.com/rss/`) can be added and refreshed.
- [ ] Mobile client connects using the API password.
- [ ] API password saved to Vaultwarden.
