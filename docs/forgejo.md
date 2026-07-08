# Forgejo

!!! success "Status — Live"
    Live in the cluster — the primary Git host and upstream for these repos.

Self-hosted Git server — the cluster's primary Git host.

| | |
|---|---|
| **URL** | `https://git.yourdomain.com` — SSH: `git@git.yourdomain.com` (same IP, see Step 4) |
| **Namespace** | `forgejo` |
| **Chart** | `forgejo` from `code.forgejo.org/forgejo-helm` (OCI), version-pinned |
| **ArgoCD Applications** | `forgejo` (chart) + `forgejo-manifests` (HTTPRoute, repos PVC, admin SealedSecret) |
| **Storage** | DB + config on `local-path` (node-pinned); repos + LFS on `nfs-storage` |
| **Auth** | Authelia OIDC (auto-registration from lldap); local `forgejo_admin` = break-glass |
| **Depends On** | Traefik (Gateway + wildcard TLS), Authelia (SSO), MetalLB (SSH LoadBalancer) |
| **Difficulty** | Intermediate |
| **Time Estimate** | 45 minutes |

Deploying via the official Helm chart (not docker-compose) keeps the Git server inside the cluster's GitOps lifecycle: ArgoCD reconciles it, an HTTPRoute gives HTTPS off the shared Gateway, a MetalLB `LoadBalancer` carries Git-over-SSH on the *same* IP, and accounts come from your lldap directory via Authelia OIDC.

!!! warning "Chart key naming"
    The Forgejo chart inherits Gitea's chart-internal `gitea.` value prefix (Forgejo forked it). This is not a bug — check the chart README at [code.forgejo.org/forgejo-helm](https://code.forgejo.org/forgejo-helm/forgejo-helm) for any value-name changes when you bump the chart.

## Storage: split the DB from the repos

The chart keeps the SQLite database **and** the git repositories under one `/data` tree by default — but they want different storage tiers:

- **SQLite cannot live on NFS.** It needs POSIX byte-range locking, which NFS does not do reliably — it *will* corrupt the database. The DB must sit on a node-local `local-path` volume.
- **Git repositories are bulk data** and are perfectly happy on NFS (git serialises writes with atomic renames, not byte-range locks).

So keep the chart's main `/data` PVC on `local-path` (small — DB, `app.ini`, avatars, attachments) and mount a *second* `nfs-storage` PVC for the repositories and LFS, relocating them with `[repository] ROOT` and `[lfs] PATH`.

!!! warning "This replaces the old single-NFS-PVC approach"
    Earlier revisions of this runbook put the whole `/data` tree on `nfs-storage`. That silently risks SQLite corruption. Always split: **DB on `local-path`, repos on `nfs-storage`.** See [Storage & Data Architecture](storage-architecture.md).

Because the DB's `local-path` PV is node-local and `ReadWriteOnce`, the pod is pinned to that node. Pin it to a worker with eMMC headroom, **off the control plane**, and ideally **off the node running your identity layer** (Authelia + lldap) — git operations (clone, pack, gc, indexing) are CPU-spiky and you don't want them contending with every app's login path. On this cluster that's `emerald` (the `workload=heavy` worker).

## Step 1: Seal the admin credentials

Forgejo needs a bootstrap admin. Even with SSO this is your **break-glass** login for when Authelia or lldap is down — so seal it, don't skip it. Don't pass the password via `--set` (it leaks into shell history and `ps`):

```bash
# Generate
FORGEJO_ADMIN_PW=$(openssl rand -base64 24)
echo "Forgejo admin password: $FORGEJO_ADMIN_PW"   # save to your password manager

kubectl create namespace forgejo

kubectl create secret generic forgejo-admin \
  --namespace forgejo \
  --from-literal=username=forgejo_admin \
  --from-literal=password="$FORGEJO_ADMIN_PW" \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > forgejo-admin-sealed.yaml

# Commit to homelab-manifests/infrastructure/forgejo/manifests/forgejo-admin-sealed.yaml
```

The chart expects the secret keys `username` and `password`. Forgejo CrashLoops until this Secret exists in the namespace.

!!! note "No separate DB secret"
    The chart dropped its bundled PostgreSQL sub-chart in v14 — for a single-instance homelab deployment Forgejo's built-in SQLite is the natural fit and needs no credential. To scale out later, deploy a shared Postgres and point `gitea.config.database` at it (see [Storage & Data Architecture](storage-architecture.md)).

## Step 2: Values

The chart renders a single-replica **Deployment** with `strategy.type: Recreate` by default — so the node-local DB volume is never double-mounted. No StatefulSet, no manual strategy override.

`values.yaml` (commit to `homelab-manifests/infrastructure/forgejo/values.yaml`):

```yaml
# Pin to the node that will hold the local-path DB (see storage rationale above).
nodeSelector:
  workload: heavy

# DB + config on local-path (node-local, POSIX-correct). Small.
persistence:
  enabled: true
  storageClass: local-path
  accessModes:
    - ReadWriteOnce
  size: 5Gi

# Repos + LFS (the bulk) on a second NFS volume, mounted into BOTH the init
# containers (which create the repo tree) and the runtime container.
extraVolumes:
  - name: forgejo-repos
    persistentVolumeClaim:
      claimName: forgejo-repos        # an nfs-storage PVC you commit under manifests/
extraContainerVolumeMounts:
  - name: forgejo-repos
    mountPath: /data-repos
extraInitVolumeMounts:
  - name: forgejo-repos
    mountPath: /data-repos

service:
  http:
    type: ClusterIP                   # HTTPS is served by the HTTPRoute, not here
  ssh:                                # see Step 4
    type: LoadBalancer
    port: 22
    annotations:
      metallb.universe.tf/loadBalancerIPs: 10.0.20.200   # Traefik's gateway IP
      metallb.universe.tf/allow-shared-ip: git-ssh-share

gitea:
  admin:
    existingSecret: forgejo-admin     # the SealedSecret from Step 1
    email: admin@yourdomain.com
  config:
    server:
      DOMAIN: git.yourdomain.com
      ROOT_URL: https://git.yourdomain.com/   # drives the OAuth callback URL
      SSH_DOMAIN: git.yourdomain.com
    # Relocate the bulk onto the NFS mount (default [repository] ROOT is on /data;
    # the deprecated LFS key is [server] LFS_CONTENT_PATH, not [lfs] PATH).
    repository:
      ROOT: /data-repos/repositories
    lfs:
      PATH: /data-repos/lfs
    # Identity comes from Authelia: allow account creation only via OAuth (Step 5).
    service:
      ALLOW_ONLY_EXTERNAL_REGISTRATION: true
    oauth2_client:
      ENABLE_AUTO_REGISTRATION: true  # auto-create a Forgejo user on first OIDC login
      USERNAME: nickname              # username from Authelia's preferred_username
      ACCOUNT_LINKING: auto           # link to an existing local user by email
```

The repos PVC (`homelab-manifests/infrastructure/forgejo/manifests/repos-pvc.yaml`):

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: forgejo-repos
  namespace: forgejo
spec:
  storageClassName: nfs-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

??? example "Bootstrap by hand (before GitOps adopts it)"

    The chart is published as an OCI artifact:

    ```bash
    helm upgrade --install forgejo oci://code.forgejo.org/forgejo-helm/forgejo \
      --version <X.Y.Z> \
      --namespace forgejo --create-namespace \
      --values values.yaml
    ```

    Pin `--version` to a current release on [code.forgejo.org/forgejo-helm](https://code.forgejo.org/forgejo-helm/forgejo-helm).

## Step 3: GitOps-managed install (recommended)

Commit the ArgoCD `Application` to `homelab-manifests/bootstrap/forgejo.yaml` so ArgoCD reconciles future changes from Git. Use the multi-source `$values` pattern (chart from upstream OCI + your `values.yaml` from this repo), plus a second `Application` for the raw manifests:

```yaml
# bootstrap/forgejo.yaml (chart Application — abridged)
spec:
  sources:
    - repoURL: code.forgejo.org/forgejo-helm   # OCI registry, no scheme prefix
      chart: forgejo
      targetRevision: 17.1.1                    # pin a version; don't track latest
      helm:
        releaseName: forgejo
        valueFiles:
          - $values/infrastructure/forgejo/values.yaml
    - repoURL: https://github.com/<you>/homelab-manifests.git
      targetRevision: HEAD
      ref: values
  syncPolicy:
    automated:
      prune: false     # don't drop the Deployment + DB PVC out from under a sync
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

The second `forgejo-manifests` Application points at `infrastructure/forgejo/manifests/` (HTTPRoute, repos PVC, admin SealedSecret). Give it `CreateNamespace=true` but **not** `ServerSideApply` — SSA plus a Gateway-API HTTPRoute is a permanent-OutOfSync trap.

!!! note "Applications register on merge"
    The app-of-apps `root.yaml` is live, so committing `bootstrap/forgejo.yaml` and merging is enough — root creates the Application on its next sync. No manual `kubectl apply`.

## Step 4: Routing — HTTPS + Git-over-SSH

**HTTPS** is an HTTPRoute on the shared Gateway's `websecure` listener — the Gateway terminates TLS with the wildcard cert, so Forgejo configures no TLS:

```yaml
# manifests/httproute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: forgejo
  namespace: forgejo
spec:
  parentRefs:
    - name: traefik
      namespace: traefik
      sectionName: websecure
  hostnames:
    - git.yourdomain.com
  rules:
    - backendRefs:
        - name: forgejo-http
          port: 3000
```

**Git-over-SSH** is raw TCP, which the HTTP Gateway can't carry. Expose it with the chart's own `service.ssh` as a MetalLB `LoadBalancer` (configured in Step 2). The trick that keeps a *single* hostname is to **share the Traefik gateway's IP**: ports 22 and 443 don't collide, so MetalLB will co-locate both Services on `10.0.20.200`, and `git@git.yourdomain.com` resolves to the same record as the web UI.

Sharing requires the **same** `allow-shared-ip` key on **both** Services. Add it to Traefik's Service in `apps/traefik/values.yaml`:

```yaml
service:
  annotations:
    metallb.universe.tf/loadBalancerIPs: 10.0.20.200
    metallb.universe.tf/allow-shared-ip: git-ssh-share   # matches Forgejo's ssh Service
```

The chart maps the Service's port 22 to container port 2222 (the rootless image's `SSH_LISTEN_PORT`); `SSH_PORT: 22` is what gets advertised in clone URLs.

!!! note "Why not a TCPRoute?"
    The Gateway API *can* route TCP, but that needs the experimental channel plus a dedicated TCP listener on the Gateway — neither is configured here (HTTP/HTTPS listeners only). The shared-IP `LoadBalancer` is the simpler path and reuses the IP you already have.

## Step 5: Single sign-on (Authelia OIDC)

Forgejo keeps its own user system, so it's an **OIDC client** of Authelia (not ForwardAuth — its Git CLI/API clients can't follow an auth redirect). Two halves:

**a. Register the client in Authelia.** Follow the *OIDC client* recipe in the [deploy pattern](apps-deploy-pattern.md): add a `forgejo` client under `configMap.identity_providers.oidc.clients` in `apps/authelia/values.yaml`, commit, and let Authelia roll. Key specifics for Forgejo:

- **Callback:** `https://git.yourdomain.com/user/oauth2/authelia/callback` — the path segment (`authelia`) **must** equal the source name you set in step (b).
- **Scopes:** include `groups`; set `claims_policy: 'default'` so the group cn lands in the ID token (Forgejo reads groups for admin mapping).
- **Token auth:** `client_secret_basic`.
- The client secret is the **pbkdf2 hash** in Authelia's values; the **plaintext** lives only in Forgejo's OAuth source (below) and your password manager — **no SealedSecret for it**.

**b. Add the OAuth source in Forgejo.** Log in as `forgejo_admin`, then **Site Administration → Authentication Sources → Add Authentication Source**:

| Field | Value |
|---|---|
| Authentication Type | **OAuth2** *(select first — it reveals the rest)* |
| Authentication Name | `authelia` *(must match the callback path above)* |
| OAuth2 Provider | **OpenID Connect** |
| Client ID (Key) | `forgejo` |
| Client Secret | *(the OIDC plaintext)* |
| OpenID Connect Auto Discovery URL | `https://auth.yourdomain.com/.well-known/openid-configuration` |
| Additional Scopes | `email profile groups` |
| Claim name providing group names for this source | `groups` |
| Group Claim value for administrator users | `homelab-admins` |
| Skip Local Two Factor Authentication | ✓ *(Authelia already enforces 2FA)* |

The group fields appear once Provider = OpenID Connect. The admin-group value keys on the lldap group **cn** (`homelab-admins`), not lldap's built-in `lldap_admin` — see the [deploy pattern](apps-deploy-pattern.md) for why.

??? tip "CLI alternative"
    ```bash
    forgejo admin auth add-oauth --name authelia --provider openidConnect \
      --key forgejo --secret '<plaintext>' \
      --auto-discover-url https://auth.yourdomain.com/.well-known/openid-configuration \
      --scopes 'openid email profile groups' \
      --group-claim-name groups --admin-group homelab-admins --skip-local-2fa
    ```

!!! warning "Groups are captured at login"
    A user's admin bit is decided from the `groups` claim **at login**. After changing someone's lldap groups, they must fully log out of *both* Forgejo and Authelia and back in.

## Step 6: First login & SSH key

1. Browse to `https://git.yourdomain.com`, log in once as `forgejo_admin` (break-glass) to confirm the cert and add the OAuth source (Step 5b).
2. Log out, then **Sign in with authelia** — you're redirected through Authelia, and a Forgejo account is auto-created from your lldap identity (admin if you're in `homelab-admins`).
3. Add your SSH key under **User Settings → SSH / GPG Keys**.

## Verification

- [ ] Pod Running and both PVCs bound:

    ```bash
    kubectl get pods,pvc -n forgejo
    # forgejo-... 1/1 Running; gitea-shared-storage (local-path) + forgejo-repos (nfs-storage) Bound
    ```

- [ ] Repos really landed on NFS (not the DB volume):

    ```bash
    kubectl exec -n forgejo deploy/forgejo -- df -hT /data /data-repos
    # /data -> local fs; /data-repos -> nfs/nfs4
    ```

- [ ] SSH answers on the shared IP:

    ```bash
    kubectl get svc -n forgejo forgejo-ssh   # EXTERNAL-IP == Traefik's 10.0.20.200
    ssh -T git@git.yourdomain.com            # "Hi there, <user>! ... does not provide shell access"
    ```

- [ ] OIDC round-trip: **Sign in with authelia** logs you in; a `homelab-admins` member lands as a Forgejo admin.
- [ ] HTTPS clone/push works; if you committed the Applications, `kubectl get application -n argocd forgejo forgejo-manifests` shows `Synced/Healthy`.
