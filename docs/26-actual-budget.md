# Runbook 26: Actual Budget

Self-hosted zero-based budgeting with end-to-end encrypted sync.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 30–45 minutes |
| **Runs On** | k3s (cluster) |
| **Depends On** | Runbook 5 (k3s), Runbook 6 (Traefik), Runbook 18 (Authelia OIDC) |

Actual Budget ([actualbudget.org](https://actualbudget.org)) is a local-first, end-to-end encrypted personal finance tool. Budget data is stored and encrypted on-device; the server only syncs encrypted blobs. ARM64 ✅ (`actualbudget/actual-server` ships multiarch). See the [install docs](https://actualbudget.org/docs/install/docker/) and [config reference](https://actualbudget.org/docs/config/) for full reference.

**Auth mode:** OIDC client (Authelia). Actual Budget has its own login flow. Do **not** use ForwardAuth.

!!! note "OIDC is in preview"
    OpenID Connect authentication is available in Actual Budget but marked as *preview*. It functions well for personal use but may have rough edges. If you prefer stability, use `password` mode and rely on Authelia ForwardAuth as a second layer instead — see the note at the bottom.

## Step 1: SealedSecret

```bash
kubectl create namespace actual-budget

kubectl create secret generic actual-budget-secrets \
  --namespace actual-budget \
  --from-literal=ACTUAL_OPENID_CLIENT_SECRET=<oidc_client_secret_plaintext> \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > actual-budget-secrets-sealed.yaml
```

Save the plaintext secret to Vaultwarden.

## Step 2: Register the OIDC Client in Authelia

In `homelab-manifests/apps/authelia/values.yaml`, add under `configMap.identity_providers.oidc.clients`:

```yaml
- client_id: actual-budget
  client_name: Actual Budget
  client_secret:
    value: <HASHED_SECRET>   # authelia crypto hash generate pbkdf2 --variant sha512
  redirect_uris:
    - https://budget.yourdomain.com/openid/callback
  scopes: [openid, profile, email]
  token_endpoint_auth_method: client_secret_basic
  grant_types: [authorization_code]
  response_types: [code]
```

Commit the updated `values.yaml` and upgrade the Authelia Helm release before deploying Actual Budget.

## Step 3: Deploy Actual Budget

Create `homelab-manifests/apps/actual-budget/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: actual-budget
  namespace: actual-budget
spec:
  replicas: 1
  selector:
    matchLabels:
      app: actual-budget
  template:
    metadata:
      labels:
        app: actual-budget
    spec:
      containers:
        - name: actual-budget
          image: actualbudget/actual-server:latest
          ports:
            - containerPort: 5006
          env:
            - name: ACTUAL_LOGIN_METHOD
              value: openid
            - name: ACTUAL_OPENID_DISCOVERY_URL
              value: https://auth.yourdomain.com
            - name: ACTUAL_OPENID_CLIENT_ID
              value: actual-budget
            - name: ACTUAL_OPENID_SERVER_HOSTNAME
              value: https://budget.yourdomain.com
            - name: ACTUAL_TRUSTED_PROXIES
              value: 10.0.0.0/8
          envFrom:
            - secretRef:
                name: actual-budget-secrets
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: actual-budget-data
```

Create `homelab-manifests/apps/actual-budget/pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: actual-budget-data
  namespace: actual-budget
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 1Gi
```

Create `homelab-manifests/apps/actual-budget/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: actual-budget
  namespace: actual-budget
spec:
  selector:
    app: actual-budget
  ports:
    - port: 5006
      targetPort: 5006
```

## Step 4: IngressRoute

Create `homelab-manifests/apps/actual-budget/ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: actual-budget
  namespace: actual-budget
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`budget.yourdomain.com`)
      kind: Rule
      services:
        - name: actual-budget
          port: 5006
  tls:
    certResolver: cloudflare
    domains:
      - main: budget.yourdomain.com
```

Commit all manifests to `homelab-manifests/apps/actual-budget/` and let ArgoCD sync.

## Step 5: First Login

Open `https://budget.yourdomain.com`. You will be redirected to Authelia login and back to Actual Budget automatically. The first OIDC user to log in becomes the server admin.

## Fallback: Password Mode

If OIDC preview is too unstable, switch to password mode and add Authelia ForwardAuth as an additional protection layer:

1. Set `ACTUAL_LOGIN_METHOD=password` instead.
2. Add the Authelia middleware to the IngressRoute.
3. Set a strong password in the Actual Budget web UI on first login.

Password mode is the default and fully stable.

## Verification

- [ ] Actual Budget pod Running:

    ```bash
    kubectl get pods -n actual-budget
    ```

- [ ] `https://budget.yourdomain.com` loads and redirects to Authelia login.
- [ ] After login, the Actual Budget file browser loads.
- [ ] Download the Actual Budget desktop or mobile app and connect it to `https://budget.yourdomain.com` to sync budgets locally.
- [ ] Credentials saved to Vaultwarden.
