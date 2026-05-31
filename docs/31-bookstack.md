# Runbook 31: BookStack

Self-hosted wiki and knowledge base with OIDC login and role-based access.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 45–60 minutes |
| **Runs On** | k3s (cluster) |
| **Depends On** | Runbook 5 (k3s), Runbook 6 (Traefik), Runbook 18 (Authelia OIDC) |

BookStack ([bookstackapp.com](https://www.bookstackapp.com)) is a structured wiki platform organised around Shelves → Books → Chapters → Pages. ARM64 ✅ (`lscr.io/linuxserver/bookstack` ships multiarch via LinuxServer.io). See the [official documentation](https://www.bookstackapp.com/docs/) for full reference.

**Auth mode:** OIDC client (Authelia). BookStack has its own user and role system. Do **not** use ForwardAuth.

BookStack requires a **MariaDB** database — both are deployed here.

## Step 1: MariaDB

Create `homelab-manifests/apps/bookstack/mariadb.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: bookstack
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookstack-db
  namespace: bookstack
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bookstack-db
  template:
    metadata:
      labels:
        app: bookstack-db
    spec:
      containers:
        - name: mariadb
          image: lscr.io/linuxserver/mariadb:latest
          ports:
            - containerPort: 3306
          env:
            - name: PUID
              value: "1000"
            - name: PGID
              value: "1000"
            - name: TZ
              value: Europe/London
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: bookstack-db-secrets
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_DATABASE
              value: bookstack
            - name: MYSQL_USER
              value: bookstack
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: bookstack-db-secrets
                  key: MYSQL_PASSWORD
          volumeMounts:
            - name: config
              mountPath: /config
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: bookstack-db
---
apiVersion: v1
kind: Service
metadata:
  name: bookstack-db
  namespace: bookstack
spec:
  selector:
    app: bookstack-db
  ports:
    - port: 3306
      targetPort: 3306
```

## Step 2: SealedSecrets

`APP_KEY` is required by BookStack for session encryption. Generate it using the official method before sealing:

```bash
# Generate APP_KEY using the LinuxServer image
APP_KEY=$(docker run --rm --entrypoint /bin/bash lscr.io/linuxserver/bookstack:latest appkey)
echo $APP_KEY  # Copy this value
```

```bash
kubectl create secret generic bookstack-db-secrets \
  --namespace bookstack \
  --from-literal=MYSQL_ROOT_PASSWORD=$(openssl rand -base64 32) \
  --from-literal=MYSQL_PASSWORD=$(openssl rand -base64 32) \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > bookstack-db-secrets-sealed.yaml

kubectl create secret generic bookstack-secrets \
  --namespace bookstack \
  --from-literal=APP_KEY=<app_key_from_above> \
  --from-literal=OIDC_CLIENT_SECRET=<oidc_client_secret_plaintext> \
  --from-literal=DB_PASSWORD=<same_MYSQL_PASSWORD_from_above> \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > bookstack-secrets-sealed.yaml
```

Save all plaintext values to Vaultwarden.

## Step 3: Register the OIDC Client in Authelia

In `homelab-manifests/apps/authelia/values.yaml`, add under `configMap.identity_providers.oidc.clients`:

```yaml
- client_id: bookstack
  client_name: BookStack
  client_secret:
    value: <HASHED_SECRET>   # authelia crypto hash generate pbkdf2 --variant sha512
  redirect_uris:
    - https://wiki.yourdomain.com/oidc/callback
  scopes: [openid, profile, email, groups]
  claims_policy: default
  token_endpoint_auth_method: client_secret_basic
  grant_types: [authorization_code]
  response_types: [code]
```

Commit and upgrade Authelia before deploying BookStack.

## Step 4: Deploy BookStack

Create `homelab-manifests/apps/bookstack/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookstack
  namespace: bookstack
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bookstack
  template:
    metadata:
      labels:
        app: bookstack
    spec:
      initContainers:
        - name: wait-for-db
          image: busybox:latest
          command: ['sh', '-c', 'until nc -z bookstack-db 3306; do sleep 2; done']
      containers:
        - name: bookstack
          image: lscr.io/linuxserver/bookstack:latest
          ports:
            - containerPort: 6875
          env:
            - name: PUID
              value: "1000"
            - name: PGID
              value: "1000"
            - name: TZ
              value: Europe/London
            - name: APP_URL
              value: https://wiki.yourdomain.com
            - name: DB_HOST
              value: bookstack-db.bookstack.svc.cluster.local
            - name: DB_DATABASE
              value: bookstack
            - name: DB_USERNAME
              value: bookstack
            - name: APP_KEY
              valueFrom:
                secretKeyRef:
                  name: bookstack-secrets
                  key: APP_KEY
            - name: AUTH_METHOD
              value: oidc
            - name: AUTH_AUTO_INITIATE
              value: "false"
            - name: OIDC_NAME
              value: Authelia
            - name: OIDC_DISPLAY_NAME_CLAIMS
              value: name
            - name: OIDC_CLIENT_ID
              value: bookstack
            - name: OIDC_ISSUER
              value: https://auth.yourdomain.com
            - name: OIDC_ISSUER_DISCOVER
              value: "true"
          envFrom:
            - secretRef:
                name: bookstack-secrets
          volumeMounts:
            - name: config
              mountPath: /config
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: bookstack-app
```

Create `homelab-manifests/apps/bookstack/pvcs.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: bookstack-db
  namespace: bookstack
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
  name: bookstack-app
  namespace: bookstack
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 5Gi
```

Create `homelab-manifests/apps/bookstack/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bookstack
  namespace: bookstack
spec:
  selector:
    app: bookstack
  ports:
    - port: 6875
      targetPort: 6875
```

## Step 5: IngressRoute

Create `homelab-manifests/apps/bookstack/ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: bookstack
  namespace: bookstack
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`wiki.yourdomain.com`)
      kind: Rule
      services:
        - name: bookstack
          port: 6875
  tls:
    certResolver: cloudflare
    domains:
      - main: wiki.yourdomain.com
```

Commit all manifests to `homelab-manifests/apps/bookstack/` and let ArgoCD sync.

## Step 6: First Login

Open `https://wiki.yourdomain.com`. With `AUTH_METHOD=oidc`, the login page shows a **Login with Authelia** button. Click it to authenticate.

The first OIDC user to log in will have the default **Viewer** role. Promote yourself to **Admin** using the default admin account (`admin@admin.com` / `password`) which is still active for local login:

1. Log in locally with `admin@admin.com` / `password`.
2. Go to **Settings → Users** and find your OIDC user.
3. Set the role to **Admin**.
4. Log out, then delete or disable the default admin account.

Save the OIDC client secret to Vaultwarden.

## Verification

- [ ] MariaDB and BookStack pods Running:

    ```bash
    kubectl get pods -n bookstack
    ```

- [ ] `https://wiki.yourdomain.com` loads the login page with an Authelia button.
- [ ] OIDC login completes and BookStack loads.
- [ ] A test page can be created.
- [ ] Default admin account deleted or disabled after your OIDC user is promoted.
