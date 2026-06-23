# Authelia + lldap

!!! success "Status — Live"
    Live in the cluster — Authelia + lldap, the SSO/OIDC backbone every other service authenticates against.

Single sign-on, two-factor authentication, and OIDC provider for all cluster services.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 2–3 hours |
| **Runs On** | k3s (cluster) |
| **Depends On** | Kubernetes (k3s), Traefik, Vaultwarden |

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

Save the plaintext `LLDAP_LDAP_USER_PASS` value to Vaultwarden. This is the bootstrap **`admin`** login for the lldap web UI — it is *not* the password Authelia binds with. Authelia uses a dedicated `authelia` service account you create in Step 3, with its own separate password.

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
  # Recreate, not RollingUpdate: the data PVC is node-local local-path RWO (see
  # below). A rolling update could momentarily schedule two pods on the same node,
  # both mounting the volume and writing users.db at once → corruption.
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: lldap
  template:
    metadata:
      labels:
        app: lldap
    spec:
      # Pin to one worker: local-path is node-local, so the pod must reschedule
      # back to the node that holds its data. Replace with your node's hostname.
      nodeSelector:
        kubernetes.io/hostname: <worker-node>
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
  # local-path (node-local), NOT nfs-storage: lldap keeps users.db (SQLite) and
  # its data-encryption key in /data, and SQLite has no reliable file locking over
  # NFS → corruption. Pair this with the Recreate strategy + nodeSelector above.
  storageClassName: local-path
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

1. Go to **Users → Create a user** and create a user named `authelia` — the service account Authelia binds with to query LDAP. Give it its **own** password and save it to Vaultwarden; this is the value you seal as `authentication.ldap.password.txt` in Step 5 (it is *not* the `admin` password).
2. Go to **Groups → lldap_password_manager** and add `authelia` to this group so it can reset user passwords.
3. Create a group named `admins` for your own user accounts (optional but recommended for OIDC access control later).
4. Create your own personal user account, and add it to `admins`.

---

## Part 2: Authelia

Authelia is deployed via the official Helm chart from `https://charts.authelia.com`. The chart is in beta — always check the [chart changelog](https://github.com/authelia/chartrepo) when upgrading. See the [official Kubernetes docs](https://www.authelia.com/integration/kubernetes/chart/) for full reference.

### Step 4: Generate the OIDC Signing Key

Authelia signs OIDC tokens with an RSA private key. Generate one now — you'll seal it together with the other secrets in Step 5:

```bash
kubectl create namespace authelia

# RSA 4096 (RS256 minimum is 2048-bit)
openssl genrsa -out /tmp/oidc.jwk.RS256.pem 4096
```

### Step 5: Seal the Authelia Secret

The chart consumes a single Secret via `secret.existingSecret`. It projects five **fixed** keys into `/secrets/internal/` and auto-generates the matching `AUTHELIA_*_FILE` env vars — several were renamed in 4.38, so let the chart manage them rather than hand-rolling `pod.env`. The OIDC JWKS key is a **sixth** key that `existingSecret` ignores; it's mounted separately via `secret.additionalSecrets` (Step 6). Seal all six into one `authelia-secrets` Secret — the `authentication.ldap.password.txt` value is the `authelia` service-account password from Step 3:

```bash
# The password you gave the `authelia` user in lldap (Step 3):
AUTHELIA_LDAP_PASS='<authelia-service-account-password>'

kubectl create secret generic authelia-secrets \
  --namespace authelia \
  --from-literal="identity_validation.reset_password.jwt.hmac.key"=$(openssl rand -base64 48) \
  --from-literal="session.encryption.key"=$(openssl rand -base64 48) \
  --from-literal="storage.encryption.key"=$(openssl rand -base64 48) \
  --from-literal="identity_providers.oidc.hmac.key"=$(openssl rand -base64 48) \
  --from-literal="authentication.ldap.password.txt=$AUTHELIA_LDAP_PASS" \
  --from-file="oidc.jwk.RS256.pem"=/tmp/oidc.jwk.RS256.pem \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > authelia-secrets-sealed.yaml

rm /tmp/oidc.jwk.RS256.pem   # the only plaintext copy
```

The five built-in keys use fixed names the chart expects — do not rename them:

| Key | Mounted at | Purpose |
|-----|-----------|---------|
| `identity_validation.reset_password.jwt.hmac.key` | `/secrets/internal/` | reset-password JWT HMAC |
| `session.encryption.key` | `/secrets/internal/` | session cookie encryption |
| `storage.encryption.key` | `/secrets/internal/` | encrypts data at rest in the DB |
| `identity_providers.oidc.hmac.key` | `/secrets/internal/` | OIDC HMAC — sealed now so enabling OIDC later needs no re-seal |
| `authentication.ldap.password.txt` | `/secrets/internal/` | the `authelia` LDAP service-account password |
| `oidc.jwk.RS256.pem` | `/secrets/authelia-secrets/` | RSA private key for OIDC token signing |

!!! warning "Never regenerate `storage.encryption.key` after first start"
    `storage.encryption.key` encrypts data at rest in Authelia's database. If you
    re-seal this Secret later and let `openssl rand` mint a **new** value, Authelia
    aborts at startup with `the configured encryption key does not appear to be
    valid for this database`. When you re-seal for *any* reason — e.g. to fix the
    LDAP password — carry the existing `storage.encryption.key` over verbatim
    rather than regenerating it. If the database is still empty (no 2FA enrolled
    yet), the quickest recovery is to delete the Authelia PVC and let it recreate a
    fresh DB keyed to the new value.

!!! tip "Mind the shell with special characters"
    If the LDAP password contains a `$`, `` ` ``, or `\`, set it with single quotes
    (as above) or `read -s AUTHELIA_LDAP_PASS` — double quotes let the shell expand
    it and you seal the wrong value, which surfaces later as an LDAP bind failure
    (`LDAP Result Code 49 "Invalid Credentials"`).

Commit `authelia-secrets-sealed.yaml` to `homelab-manifests/apps/authelia/`.

### Step 6: Install Authelia via Helm

Add the chart repo and inspect the values reference:

```bash
helm repo add authelia https://charts.authelia.com
helm repo update
helm show values authelia/authelia > authelia-values-reference.yaml
```

Create `homelab-manifests/apps/authelia/values.yaml`. Replace `yourdomain.com`, `dc=yourdomain,dc=com`, and `<worker-node>` with your actual values:

```yaml
pod:
  # StatefulSet, NOT the chart default (DaemonSet). A DaemonSet runs one Authelia
  # per node, each with its own in-memory session store (login loops) all racing
  # for the single RWO PV. A single-replica StatefulSet rolls serially, so it
  # can't double-mount the local-path volume.
  kind: StatefulSet
  replicas: 1
  selectors:
    # Pin to the same worker as the PVC: local-path is node-local RWO, so the pod
    # must reschedule back to its data. Replace with your node's hostname.
    nodeSelector:
      kubernetes.io/hostname: <worker-node>

service:
  # Authelia's canonical port (chart default is 80). Keeps the HTTPRoute backend
  # and the ForwardAuth middleware address consistent.
  port: 9091

persistence:
  # Persist /config — it holds db.sqlite3 (TOTP secrets, WebAuthn, OAuth2
  # consents). local-path (node-local), never nfs-storage: SQLite corrupts on NFS.
  enabled: true
  storageClass: local-path
  accessModes:
    - ReadWriteOnce
  size: 1Gi

secret:
  # The chart projects five built-in keys from authelia-secrets into
  # /secrets/internal/ and auto-wires the AUTHELIA_*_FILE env vars. The OIDC JWKS
  # key is a sixth key existingSecret ignores, so mount the SAME secret again via
  # additionalSecrets to expose it at /secrets/authelia-secrets/oidc.jwk.RS256.pem.
  existingSecret: authelia-secrets
  additionalSecrets:
    authelia-secrets:
      items:
        - key: oidc.jwk.RS256.pem
          path: oidc.jwk.RS256.pem

configMap:
  theme: light
  log:
    level: info

  authentication_backend:
    ldap:
      enabled: true            # required (chart default false)
      implementation: lldap
      address: ldap://lldap.lldap.svc.cluster.local:3890
      base_dn: dc=yourdomain,dc=com
      user: uid=authelia,ou=people,dc=yourdomain,dc=com
      # password comes from authentication.ldap.password.txt in authelia-secrets

  access_control:
    default_policy: deny
    rules:
      - domain: auth.yourdomain.com
        policy: bypass
      # two_factor by default (secure-by-default). A two_factor rule is ALSO
      # what makes Authelia expose 2FA device registration at all — see Step 8.
      # Relax per-app with a bypass/one_factor rule placed ABOVE this catch-all.
      - domain: "*.yourdomain.com"
        policy: two_factor

  session:
    cookies:
      # Scoped to the parent domain so one SSO session spans every subdomain. The
      # chart builds authelia_url from subdomain + domain.
      - subdomain: auth
        domain: yourdomain.com
        default_redirection_url: https://yourdomain.com

  storage:
    local:
      enabled: true            # SQLite at /config/db.sqlite3 (chart default false)

  notifier:
    filesystem:
      enabled: true            # writes to /config/notification.txt (no SMTP yet)

  identity_providers:
    oidc:
      # Keep DISABLED until you add your first OIDC client (see OIDC Client
      # Setup below). Authelia refuses to start if OIDC is enabled with an empty
      # clients list. The jwks key below stays wired and ready for that moment.
      enabled: false
      claims_policies:
        default:
          id_token: [email, email_verified, name, preferred_username, groups]
      jwks:
        - key_id: default
          algorithm: RS256
          use: sig
          key:
            # Mounted via secret.additionalSecrets above, not /secrets/internal.
            path: /secrets/authelia-secrets/oidc.jwk.RS256.pem
      clients: []   # OIDC clients are added per-app in each app's runbook

ingress:
  enabled: false   # we route via an HTTPRoute (Step 7)
```

Deploy:

```bash
helm upgrade --install authelia authelia/authelia \
  --version <X.Y.Z> \
  --namespace authelia --create-namespace \
  --values homelab-manifests/apps/authelia/values.yaml
```

Pin `--version` to the latest release listed on [charts.authelia.com](https://charts.authelia.com/).

!!! warning "Leave OIDC disabled until the first client exists"
    The `oidc` block above sets `enabled: false`. Authelia 4.39 aborts at startup
    if the provider is enabled with an empty `clients` list:
    `identity_providers: oidc: option 'clients' must have one or more clients
    configured`. Nothing needs OIDC at bring-up — ForwardAuth covers the
    login-less apps — so the provider stays off until your first OIDC-integrated
    app. Flip `enabled: true` in the **same** change that adds the first client
    (see [OIDC Client Setup](#oidc-client-setup-per-app)).

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

Open `https://auth.yourdomain.com` and log in with your personal lldap user. With a single factor you land on the portal's "Authenticated" screen — Authelia does **not** auto-prompt for TOTP. Enrol it from the settings page: go to **`auth.yourdomain.com/settings/two-factor-authentication`** and click **Add** under **One-Time Password**.

!!! warning "2FA enrolment needs a `two_factor` rule"
    Authelia only exposes device registration when at least one `access_control`
    rule uses `two_factor`. With only `bypass`/`one_factor` rules, the settings
    page reports *"There are no protected applications that require a second factor
    method"* and refuses to register anything — the catch-all in Step 6 is set to
    `two_factor` for exactly this reason.

!!! note "Filesystem notifier — read the verification message from disk"
    Before showing the QR, Authelia "emails" you a link to verify your identity.
    The filesystem notifier writes it to a file instead of sending mail, so
    retrieve it from the running pod:
    ```bash
    kubectl -n authelia exec authelia-0 -- cat /config/notification.txt
    ```
    Follow the link, then scan the QR with a dedicated TOTP authenticator app
    (e.g. Ente Auth, Aegis) — keeping the second factor out of the same vault as
    your passwords. The same notifier applies to any future password-reset message
    until `notifier.smtp` is configured.

Once logged in, any service whose HTTPRoute attaches an `authelia-forwardauth` middleware (via an `ExtensionRef` filter) will redirect unauthenticated requests to this portal.

## OIDC Client Setup (Per-App)

Each app that has its own user system gets an OIDC client entry added to the `configMap.identity_providers.oidc.clients` list in `values.yaml`.

!!! warning "Enable the provider with your first client"
    The provider ships disabled (`enabled: false`, Step 6). When you add the
    **first** client, set `identity_providers.oidc.enabled: true` in the same
    change — enabling it with an empty `clients` list crashes Authelia, and a
    populated list with the provider still disabled does nothing.

The general pattern for a confidential client is:

```yaml
clients:
  - client_id: 'appname'
    client_name: 'App Display Name'
    client_secret: '<HASHED_SECRET>'   # inline pbkdf2 hash — a string, NOT a value: sub-map. gitleaks:allow
    authorization_policy: 'two_factor'
    redirect_uris:
      - 'https://appname.yourdomain.com/oauth2/callback'   # Varies per app — check app docs
    scopes: ['openid', 'profile', 'email', 'groups']
    claims_policy: 'default'
    token_endpoint_auth_method: 'client_secret_basic'
    grant_types: ['authorization_code']
    response_types: ['code']
```

!!! note "claims_policy required for groups in the ID token"
    As of Authelia 4.39, group membership is **not** included in the ID token by default. A client that reads groups from the **ID token** must set `claims_policy: default` to reference the claims policy defined in `configMap.identity_providers.oidc.claims_policies` — without it, apps like Mealie (`OIDC_ADMIN_GROUP`) and Donetick (`admin_groups`) authenticate users but never see their groups. Apps that instead read groups from the **userinfo endpoint** (e.g. Argo CD with `enableUserInfoGroups`) get them from the `groups` scope alone and don't need the policy.

!!! warning "Match the lldap group cn, not lldap's internal name"
    Authelia forwards each lldap group's **cn verbatim** in the `groups` claim. App RBAC that maps a group to a role must use that exact name — this cluster maps a dedicated **`homelab-admins`** group (created in lldap), **not** lldap's built-in `lldap_admin`. Group membership is captured at login, so re-login after any group change. (Argo CD's OIDC bring-up hit exactly this — the first mapping used the wrong group name and every SSO login landed with no permissions.)

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
- [ ] Log in with your lldap user — session is established; enrol TOTP at `auth.yourdomain.com/settings/two-factor-authentication`.
- [ ] `https://lldap.yourdomain.com` redirects to Authelia login before showing the lldap UI.
- [ ] A service using the ForwardAuth middleware (e.g., Homepage) redirects to `auth.yourdomain.com` for unauthenticated requests.
