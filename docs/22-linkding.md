# Runbook 22: linkding

Self-hosted bookmark manager with browser extension support and OIDC login.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 30–45 minutes |
| **Runs On** | k3s (cluster) |
| **Depends On** | Runbook 5 (k3s), Runbook 6 (Traefik), Runbook 18 (Authelia OIDC) |

linkding ([github.com/sissbruecker/linkding](https://github.com/sissbruecker/linkding)) is a minimal bookmark manager with tag support, a browser extension for one-click saving, and a REST API. ARM64 ✅ (`sissbruecker/linkding` ships multiarch builds). See the [official install docs](https://linkding.link/installation/) and [options reference](https://linkding.link/options/) for full reference.

**Auth mode:** OIDC client (Authelia). linkding has its own user system — do **not** use ForwardAuth.

## Step 1: SealedSecret

Create `homelab-manifests/apps/linkding/sealed-secret.yaml`. The secret carries the OIDC client secret and initial superuser password:

```bash
kubectl create namespace linkding

kubectl create secret generic linkding-secrets \
  --namespace linkding \
  --from-literal=LD_SUPERUSER_PASSWORD=<initial_admin_password> \
  --from-literal=OIDC_RP_CLIENT_SECRET=<oidc_client_secret_plaintext> \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > linkding-secrets-sealed.yaml
```

Save both values to Vaultwarden.

## Step 2: Register the OIDC Client in Authelia

In `homelab-manifests/apps/authelia/values.yaml`, add a client entry under `configMap.identity_providers.oidc.clients`:

```yaml
- client_id: linkding
  client_name: linkding
  client_secret:
    value: <HASHED_SECRET>   # authelia crypto hash generate pbkdf2 --variant sha512
  redirect_uris:
    - https://bookmarks.yourdomain.com/oidc/callback/
  scopes: [openid, profile, email]
  token_endpoint_auth_method: client_secret_basic
  grant_types: [authorization_code]
  response_types: [code]
```

!!! warning "Trailing slash"
    The linkding OIDC callback URL requires a trailing slash: `/oidc/callback/`.

Commit the updated `values.yaml` and upgrade the Authelia Helm release before deploying linkding.

## Step 3: Deploy linkding

Create `homelab-manifests/apps/linkding/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: linkding
  namespace: linkding
spec:
  replicas: 1
  selector:
    matchLabels:
      app: linkding
  template:
    metadata:
      labels:
        app: linkding
    spec:
      containers:
        - name: linkding
          image: sissbruecker/linkding:latest
          ports:
            - containerPort: 9090
          env:
            - name: LD_SUPERUSER_NAME
              value: admin
            - name: LD_ENABLE_OIDC
              value: "True"
            - name: OIDC_OP_AUTHORIZATION_ENDPOINT
              value: https://auth.yourdomain.com/api/oidc/authorization
            - name: OIDC_OP_TOKEN_ENDPOINT
              value: https://auth.yourdomain.com/api/oidc/token
            - name: OIDC_OP_USER_ENDPOINT
              value: https://auth.yourdomain.com/api/oidc/userinfo
            - name: OIDC_OP_JWKS_ENDPOINT
              value: https://auth.yourdomain.com/jwks.json
            - name: OIDC_RP_CLIENT_ID
              value: linkding
            - name: OIDC_RP_SIGN_ALGO
              value: RS256
            - name: OIDC_USE_PKCE
              value: "True"
            - name: LD_CSRF_TRUSTED_ORIGINS
              value: https://bookmarks.yourdomain.com
            - name: LD_USE_X_FORWARDED_HOST
              value: "True"
          envFrom:
            - secretRef:
                name: linkding-secrets
          volumeMounts:
            - name: data
              mountPath: /etc/linkding/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: linkding-data
```

Create `homelab-manifests/apps/linkding/pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: linkding-data
  namespace: linkding
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 1Gi
```

Create `homelab-manifests/apps/linkding/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: linkding
  namespace: linkding
spec:
  selector:
    app: linkding
  ports:
    - port: 9090
      targetPort: 9090
```

## Step 4: IngressRoute

Create `homelab-manifests/apps/linkding/ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: linkding
  namespace: linkding
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`bookmarks.yourdomain.com`)
      kind: Rule
      services:
        - name: linkding
          port: 9090
  tls:
    certResolver: cloudflare
    domains:
      - main: bookmarks.yourdomain.com
```

No Authelia middleware here — linkding's OIDC integration handles authentication itself.

Commit all manifests to `homelab-manifests/apps/linkding/` and let ArgoCD sync.

## Step 5: First Login

Open `https://bookmarks.yourdomain.com`. You will see a **Login with OIDC** button alongside the standard login form. Click it to authenticate via Authelia.

The `admin` user created from `LD_SUPERUSER_NAME` + `LD_SUPERUSER_PASSWORD` is available for local login as a fallback. Save those credentials to Vaultwarden if you haven't already.

## Step 6: Browser Extension

Install the linkding browser extension from your browser's extension store. Configure it with:

- **Server URL**: `https://bookmarks.yourdomain.com`
- **API Token**: Profile → Generate API token in the linkding UI

The extension allows one-click bookmark saving with tag pre-population from the current page title.

## Verification

- [ ] linkding pod Running:

    ```bash
    kubectl get pods -n linkding
    ```

- [ ] `https://bookmarks.yourdomain.com` loads the linkding UI.
- [ ] Clicking **Login with OIDC** redirects to `auth.yourdomain.com` and back successfully.
- [ ] A test bookmark can be saved and retrieved.
- [ ] Credentials saved to Vaultwarden.
