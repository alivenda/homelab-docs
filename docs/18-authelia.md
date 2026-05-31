# Runbook 18: Authelia + lldap

Single sign-on, two-factor authentication, and OIDC provider for all cluster services.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 2–3 hours |
| **Runs On** | k3s (cluster) |
| **Depends On** | Runbook 5 (k3s), Runbook 6 (Traefik), Runbook 7 (Vaultwarden) |

This runbook deploys two services:

- **lldap** — a lightweight LDAP server that stores your users and groups. It is the user database Authelia queries when someone logs in.
- **Authelia** — the authentication gateway. It sits behind Traefik and enforces who can access which services, handling 2FA and acting as an OIDC provider.

## ForwardAuth vs OIDC — Read This First

Authelia protects services in two fundamentally different ways. Using the wrong mode causes double-login prompts and breaks API/sync clients:

| Mode | When to use | How it works |
|------|-------------|-------------|
| **ForwardAuth** | Apps with **no** login page of their own | Traefik intercepts every request, asks Authelia "is this user authenticated?", and either passes the request through or redirects to the Authelia login portal |
| **OIDC** | Apps with **their own** user system (Nextcloud, Forgejo, Vikunja, etc.) | The app redirects to Authelia for login, receives a token, and manages its own session — Traefik is not involved in the auth check |

**Do not put ForwardAuth in front of Nextcloud, Forgejo, or Paperless.** Their API clients (desktop sync, git CLI, mobile apps) send credentials directly and cannot handle an intermediate redirect. Configure those apps as OIDC clients instead — each app's runbook covers its own OIDC setup.

**Apps that use ForwardAuth in this stack:** Homepage, any service without its own login screen.

**Apps that use OIDC in this stack:** Nextcloud, Forgejo, Paperless-ngx, Vikunja, Actual Budget, Mealie, Audiobookshelf, BookStack, and any app with a built-in user system.

---

## Part 1: lldap

lldap is the lightweight LDAP backend Authelia uses as its user store. Deploy it first — Authelia depends on it being reachable. The official image (`ghcr.io/lldap/lldap`) ships multiarch including ARM64. See the [lldap repository](https://github.com/lldap/lldap) for reference.

### Step 1: Seal lldap Credentials

Generate the required secrets and seal them. All three values must be random — they are used for JWT signing and data encryption inside lldap.

```bash
kubectl create namespace lldap

kubectl create secret generic lldap-secrets \
  --namespace lldap \
  --from-literal=LLDAP_JWT_SECRET=$(openssl rand -base64 32) \
  --from-literal=LLDAP_KEY_SEED=$(openssl rand -base64 32) \
  --from-literal=LLDAP_LDAP_USER_PASS=$(openssl rand -base64 24) \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > lldap-secrets-sealed.yaml
```

Save the plaintext `LLDAP_LDAP_USER_PASS` value to Vaultwarden — you will need it when configuring Authelia.

Commit `lldap-secrets-sealed.yaml` to `homelab-manifests/apps/lldap/`.

### Step 2: Deploy lldap

Create `homelab-manifests/apps/lldap/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lldap
  namespace: lldap
spec:
  replicas: 1
  selector:
    matchLabels:
      app: lldap
  template:
    metadata:
      labels:
        app: lldap
    spec:
      containers:
        - name: lldap
          image: ghcr.io/lldap/lldap:latest-alpine
          ports:
            - containerPort: 3890   # LDAP
            - containerPort: 17170  # Web UI
          env:
            - name: LLDAP_HTTP_URL
              value: https://lldap.yourdomain.com
            - name: LLDAP_LDAP_BASE_DN
              value: dc=yourdomain,dc=com
          envFrom:
            - secretRef:
                name: lldap-secrets
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: lldap-data
```

Create `homelab-manifests/apps/lldap/pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lldap-data
  namespace: lldap
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 1Gi
```

Create `homelab-manifests/apps/lldap/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lldap
  namespace: lldap
spec:
  selector:
    app: lldap
  ports:
    - name: ldap
      port: 3890
      targetPort: 3890
    - name: http
      port: 17170
      targetPort: 17170
```

Create the HTTPRoute for the web UI, protected by Authelia ForwardAuth. Under the Gateway API the `Middleware` lives in lldap's own namespace (an `ExtensionRef` filter can't cross namespaces) and points back at the Authelia service:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: authelia-forwardauth
  namespace: lldap
spec:
  forwardAuth:
    address: http://authelia.authelia.svc.cluster.local:9091/api/authz/forward-auth
    trustForwardHeader: true
    authResponseHeaders: [Remote-User, Remote-Groups, Remote-Name, Remote-Email]
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: lldap
  namespace: lldap
spec:
  parentRefs:
    - name: traefik
      namespace: traefik
      sectionName: websecure
  hostnames:
    - lldap.yourdomain.com
  rules:
    - filters:
        - type: ExtensionRef
          extensionRef:
            group: traefik.io
            kind: Middleware
            name: authelia-forwardauth
      backendRefs:
        - name: lldap
          port: 17170
```

!!! note
    Authelia must be running (Part 2) and the `Middleware` must exist before this route attaches with auth. The ForwardAuth middleware is copied into each protected namespace — see [Deploying an App](apps-deploy-pattern.md).

### Step 3: Create the Authelia Service Account in lldap

Once lldap is running, open its web UI at `http://<node-ip>:17170` (via `kubectl port-forward` before Traefik is wired up):

```bash
kubectl port-forward -n lldap svc/lldap 17170:17170
```

Open `http://localhost:17170`. Log in with username `admin` and the `LLDAP_LDAP_USER_PASS` you saved to Vaultwarden.

1. Go to **Users → Create a user** and create a user named `authelia`. This is the service account Authelia uses to query LDAP.
2. Go to **Groups → lldap_password_manager** and add `authelia` to this group so it can reset user passwords.
3. Create a group named `admins` for your own user accounts (optional but recommended for OIDC access control later).
4. Create your own personal user account.

---

## Part 2: Authelia

Authelia is deployed via the official Helm chart from `https://charts.authelia.com`. The chart is in beta — always check the [chart changelog](https://github.com/authelia/chartrepo) when upgrading. See the [official Kubernetes docs](https://www.authelia.com/integration/kubernetes/chart/) for full reference.

### Step 4: Generate the OIDC Signing Key

Authelia requires an RSA private key for signing OIDC tokens. Generate one and seal it as a secret:

```bash
openssl genrsa -out jwk-rsa-private.pem 4096

kubectl create namespace authelia

kubectl create secret generic authelia-oidc-jwk \
  --namespace authelia \
  --from-file=private.pem=jwk-rsa-private.pem \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > authelia-oidc-jwk-sealed.yaml

rm jwk-rsa-private.pem
```

### Step 5: Seal Authelia Secrets

```bash
kubectl create secret generic authelia-secrets \
  --namespace authelia \
  --from-literal=JWT_TOKEN=$(openssl rand -base64 48) \
  --from-literal=SESSION_ENCRYPTION_KEY=$(openssl rand -base64 48) \
  --from-literal=STORAGE_ENCRYPTION_KEY=$(openssl rand -base64 48) \
  --from-literal=LDAP_PASSWORD=<LLDAP_LDAP_USER_PASS_FROM_VAULTWARDEN> \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > authelia-secrets-sealed.yaml
```

Commit both sealed secrets to `homelab-manifests/apps/authelia/`.

### Step 6: Install Authelia via Helm

Add the chart repo and inspect the values reference:

```bash
helm repo add authelia https://charts.authelia.com
helm repo update
helm show values authelia/authelia > authelia-values-reference.yaml
```

Create `homelab-manifests/apps/authelia/values.yaml`. Replace `yourdomain.com`, `dc=yourdomain,dc=com`, and `dc=com` with your actual domain components:

```yaml
pod:
  replicas: 1
  env:
    - name: AUTHELIA_JWT_SECRET_FILE
      value: /secrets/authelia/JWT_TOKEN
    - name: AUTHELIA_SESSION_SECRET_FILE
      value: /secrets/authelia/SESSION_ENCRYPTION_KEY
    - name: AUTHELIA_STORAGE_ENCRYPTION_KEY_FILE
      value: /secrets/authelia/STORAGE_ENCRYPTION_KEY
    - name: AUTHELIA_AUTHENTICATION_BACKEND_LDAP_PASSWORD_FILE
      value: /secrets/authelia/LDAP_PASSWORD
  extraVolumes:
    - name: authelia-secrets
      secret:
        secretName: authelia-secrets
    - name: jwk-rsa
      secret:
        secretName: authelia-oidc-jwk
  extraVolumeMounts:
    - name: authelia-secrets
      mountPath: /secrets/authelia
      readOnly: true
    - name: jwk-rsa
      mountPath: /secrets/jwk-rsa
      readOnly: true

configMap:
  log:
    level: info

  authentication_backend:
    ldap:
      implementation: lldap
      address: ldap://lldap.lldap.svc.cluster.local:3890
      base_dn: dc=yourdomain,dc=com
      user: uid=authelia,ou=people,dc=yourdomain,dc=com

  access_control:
    default_policy: deny
    rules:
      - domain: auth.yourdomain.com
        policy: bypass
      - domain: "*.yourdomain.com"
        policy: one_factor

  session:
    cookies:
      - name: authelia_session
        domain: yourdomain.com
        authelia_url: https://auth.yourdomain.com
        default_redirection_url: https://yourdomain.com

  storage:
    local:
      path: /config/db.sqlite3

  notifier:
    filesystem:
      filename: /tmp/notification.txt

  identity_providers:
    oidc:
      enabled: true
      jwks:
        - key_id: default
          algorithm: RS256
          key:
            path: /secrets/jwk-rsa/private.pem
      claims_policies:
        default:
          id_token: [email, email_verified, name, preferred_username, groups]
      clients: []   # OIDC clients are added per-app in each app's runbook

ingress:
  enabled: false   # We use an HTTPRoute below
```

Deploy:

```bash
helm upgrade --install authelia authelia/authelia \
  --version <X.Y.Z> \
  --namespace authelia --create-namespace \
  --values homelab-manifests/apps/authelia/values.yaml
```

Pin `--version` to the latest release listed on [charts.authelia.com](https://charts.authelia.com/).

### Step 7: HTTPRoute and ForwardAuth Middleware

The Authelia portal itself is **not** behind ForwardAuth (it *is* the auth). Create `homelab-manifests/apps/authelia/manifests/httproute.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: authelia
  namespace: authelia
spec:
  parentRefs:
    - name: traefik
      namespace: traefik
      sectionName: websecure
  hostnames:
    - auth.yourdomain.com
  rules:
    - backendRefs:
        - name: authelia
          port: 9091
```

Every *other* protected service gets a ForwardAuth `Middleware` **in its own namespace**, referenced from its `HTTPRoute` with an `ExtensionRef` filter. Under the Gateway API an `ExtensionRef` can't cross namespaces, so — unlike the old `authelia@kubernetescrd` cross-namespace reference — this `Middleware` is copied into each protected app's namespace (its `address` still points back at the central Authelia service):

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: authelia-forwardauth
  namespace: <app-namespace>
spec:
  forwardAuth:
    address: http://authelia.authelia.svc.cluster.local:9091/api/authz/forward-auth
    trustForwardHeader: true
    authResponseHeaders:
      - Remote-User
      - Remote-Groups
      - Remote-Email
      - Remote-Name
```

See [Deploying an App](apps-deploy-pattern.md) for the per-app HTTPRoute + `ExtensionRef` wiring. Commit all manifests to `homelab-manifests/apps/authelia/` and let ArgoCD sync.

!!! note "ExtensionRef prerequisite"
    Traefik's Kubernetes IngressRoute CRD provider must stay enabled for `ExtensionRef` middlewares to resolve — moving to the Gateway API doesn't remove that requirement.

### Step 8: First Login

Open `https://auth.yourdomain.com`. Log in with your personal user account from lldap. You should be prompted to set up TOTP (use Vaultwarden's built-in TOTP generator and save the secret).

Once logged in, any service whose HTTPRoute attaches an `authelia-forwardauth` middleware (via an `ExtensionRef` filter) will redirect unauthenticated requests to this portal.

## OIDC Client Setup (Per-App)

Each app that has its own user system gets an OIDC client entry added to the `configMap.identity_providers.oidc.clients` list in `values.yaml`. The general pattern for a confidential client is:

```yaml
clients:
  - client_id: appname
    client_name: App Display Name
    client_secret:
      value: <HASHED_SECRET>   # Generate: authelia crypto hash generate pbkdf2 --variant sha512
    redirect_uris:
      - https://appname.yourdomain.com/oauth2/callback   # Varies per app — check app docs
    scopes: [openid, profile, email, groups]
    claims_policy: default
    token_endpoint_auth_method: client_secret_basic
    grant_types: [authorization_code]
    response_types: [code]
```

!!! note "claims_policy required for groups"
    As of Authelia 4.39, group membership is **not** included in the ID token by default. Any client that requests the `groups` scope must also set `claims_policy: default` to reference the claims policy defined in `configMap.identity_providers.oidc.claims_policies`. Without this, apps like Mealie (`OIDC_ADMIN_GROUP`) and Donetick (`admin_groups`) will authenticate users but never recognise their group memberships.

!!! warning "Hashed client secrets"
    As of Authelia 4.38, client secrets must be stored as hashes in the config. Generate with:
    ```bash
    authelia crypto hash generate pbkdf2 --variant sha512
    ```
    The plaintext secret goes into the app's own config — save both to Vaultwarden.

Each app's runbook covers its specific `redirect_uris` and configuration. Do not attempt to configure OIDC for all apps here — add clients incrementally as you deploy each service.

## Verification

- [ ] lldap pod Running:

    ```bash
    kubectl get pods -n lldap
    ```

- [ ] Authelia pod Running:

    ```bash
    kubectl get pods -n authelia
    ```

- [ ] `https://auth.yourdomain.com` loads the Authelia login portal.
- [ ] Log in with your lldap user — session is established, TOTP prompt appears.
- [ ] `https://lldap.yourdomain.com` redirects to Authelia login before showing the lldap UI.
- [ ] A service using the ForwardAuth middleware (e.g., Homepage after Runbook 20) redirects to `auth.yourdomain.com` for unauthenticated requests.
