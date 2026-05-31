# Runbook 30: Mealie

Self-hosted recipe manager with meal planning, grocery lists, and OIDC login.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 30–45 minutes |
| **Runs On** | k3s (cluster) |
| **Depends On** | Runbook 5 (k3s), Runbook 6 (Traefik), Runbook 18 (Authelia OIDC) |

Mealie ([mealie.io](https://mealie.io)) is a recipe manager with ingredient parsing, meal planning, grocery list generation, and a scraper for importing recipes from URLs. ARM64 ✅ (`ghcr.io/mealie-recipes/mealie` ships multiarch). See the [official docs](https://docs.mealie.io) for full reference.

**Auth mode:** OIDC client (Authelia). Mealie has its own user system with group-based access. Do **not** use ForwardAuth.

## Step 1: SealedSecret

```bash
kubectl create namespace mealie

kubectl create secret generic mealie-secrets \
  --namespace mealie \
  --from-literal=OIDC_CLIENT_SECRET=<oidc_client_secret_plaintext> \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > mealie-secrets-sealed.yaml
```

Save the plaintext secret to Vaultwarden.

## Step 2: Register the OIDC Client in Authelia

In `homelab-manifests/apps/authelia/values.yaml`, add under `configMap.identity_providers.oidc.clients`:

```yaml
- client_id: mealie
  client_name: Mealie
  client_secret:
    value: <HASHED_SECRET>   # authelia crypto hash generate pbkdf2 --variant sha512
  redirect_uris:
    - https://recipes.yourdomain.com/login
    - https://recipes.yourdomain.com/login?direct=1
  scopes: [openid, profile, email, groups]
  token_endpoint_auth_method: client_secret_basic
  grant_types: [authorization_code]
  response_types: [code]
```

!!! note "PKCE required"
    Mealie v2 requires a confidential client (with a client secret) and uses the Authorization Code flow. The two redirect URIs are both required — the second is used when `OIDC_AUTO_REDIRECT=true`.

Commit the updated `values.yaml` and upgrade the Authelia Helm release before deploying Mealie.

## Step 3: Deploy Mealie

Create `homelab-manifests/apps/mealie/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mealie
  namespace: mealie
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mealie
  template:
    metadata:
      labels:
        app: mealie
    spec:
      containers:
        - name: mealie
          image: ghcr.io/mealie-recipes/mealie:latest
          ports:
            - containerPort: 9000
          env:
            - name: PUID
              value: "1000"
            - name: PGID
              value: "1000"
            - name: TZ
              value: Europe/London
            - name: BASE_URL
              value: https://recipes.yourdomain.com
            - name: ALLOW_SIGNUP
              value: "false"
            - name: ALLOW_PASSWORD_LOGIN
              value: "true"
            - name: OIDC_AUTH_ENABLED
              value: "true"
            - name: OIDC_SIGNUP_ENABLED
              value: "true"
            - name: OIDC_CONFIGURATION_URL
              value: https://auth.yourdomain.com/.well-known/openid-configuration
            - name: OIDC_CLIENT_ID
              value: mealie
            - name: OIDC_PROVIDER_NAME
              value: Authelia
            - name: OIDC_REMEMBER_ME
              value: "true"
          envFrom:
            - secretRef:
                name: mealie-secrets
          volumeMounts:
            - name: data
              mountPath: /app/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: mealie-data
```

Create `homelab-manifests/apps/mealie/pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mealie-data
  namespace: mealie
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 5Gi
```

Create `homelab-manifests/apps/mealie/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mealie
  namespace: mealie
spec:
  selector:
    app: mealie
  ports:
    - port: 9000
      targetPort: 9000
```

## Step 4: IngressRoute

Create `homelab-manifests/apps/mealie/ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: mealie
  namespace: mealie
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`recipes.yourdomain.com`)
      kind: Rule
      services:
        - name: mealie
          port: 9000
  tls:
    certResolver: cloudflare
    domains:
      - main: recipes.yourdomain.com
```

No Authelia middleware — Mealie's OIDC handles authentication.

Commit all manifests to `homelab-manifests/apps/mealie/` and let ArgoCD sync.

## Step 5: First Login

Open `https://recipes.yourdomain.com`. You will see a **Login with Authelia** button. OIDC login creates a Mealie user automatically on first sign-in.

The default admin credentials (`changeme@example.com` / `MyPassword`) are active until you log in via OIDC and promote your OIDC user to admin via the Mealie admin panel (Settings → Users → change role to Admin).

Once promoted, you can disable `ALLOW_PASSWORD_LOGIN` in the Deployment env vars to hide the local login form.

## Verification

- [ ] Mealie pod Running:

    ```bash
    kubectl get pods -n mealie
    ```

- [ ] `https://recipes.yourdomain.com` loads the login page with the Authelia button.
- [ ] OIDC login completes and Mealie home screen loads.
- [ ] A recipe URL can be imported via the scraper.
- [ ] Credentials saved to Vaultwarden.
