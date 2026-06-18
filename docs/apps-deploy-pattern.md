# Deploying an App — the GitOps Pattern

Every user-facing service on the cluster is deployed the same way. This is that procedure, written **once**. Per-app specifics live either in the [App Catalog](apps-catalog.md) (simple services — just the deltas) or in a dedicated runbook (services with real complexity: Arr stack, Authelia, Immich, Home Assistant, Ollama, ntfy).

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Runs On** | k3s (cluster) |
| **Depends On** | A working Traefik `Gateway` + wildcard TLS (R6), and Authelia (R18) for auth |

!!! note "Source of truth is the manifest, not this page"
    The deployed state lives in `homelab-manifests/apps/<app>/`. This runbook documents the *procedure*; it deliberately does not restate per-app YAML. App-specific values (chart version, env, hostname) belong in the manifest plus the catalog row — not duplicated into prose that drifts.

## The shape of an app

Each app is one directory in `homelab-manifests/apps/<app>/` plus one or two ArgoCD `Application`s in `bootstrap/`:

- **Workload** — a Helm chart (preferred when upstream ships a good one) *or* raw manifests (`deployment.yaml` + `service.yaml` + `pvc.yaml`).
- **`<app>-manifests` Application** — the namespaced extras ArgoCD applies from `apps/<app>/manifests/`: the `HTTPRoute`, any `SealedSecret`, and (for ForwardAuth apps) a `Middleware`.

```text
apps/<app>/
├── values.yaml              # Helm mode only
└── manifests/
    ├── httproute.yaml
    ├── <app>-sealed.yaml    # if it needs secrets
    └── middleware.yaml      # ForwardAuth apps only
```

## Step 1 — Choose the workload mode

- **Helm chart** when upstream maintains one (e.g. Vaultwarden). The `Application` uses the multi-source `$values` pattern: chart from the upstream Helm repo + `valueFiles: [$values/apps/<app>/values.yaml]`. Pin `targetRevision` and add a `# renovate:` comment so Renovate tracks the chart.
- **Raw manifests** when there's no good chart — most simple apps (linkding, Mealie, Uptime Kuma). Commit a `Deployment` + `Service` + `PVC` to `apps/<app>/manifests/`. **Pin the image tag — never `:latest`.**

## Step 2 — Secrets (SealedSecret)

Anything sensitive (OIDC client secret, admin password, API token) goes in a `SealedSecret` so it's safe to commit. Seal against the cluster's controller:

```bash
kubectl create secret generic <app>-secrets \
  --namespace <app> \
  --from-literal=<KEY>=<plaintext> \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > apps/<app>/manifests/<app>-sealed.yaml
```

Save the plaintext to Vaultwarden as well. The controller's signing key is itself backed up — see [Backups & DR](10-backups.md).

## Step 3 — Storage

See **[Storage & Data Architecture](storage-architecture.md)** for the full rationale and the per-tier rules; the short version:

| Use | StorageClass | Notes |
|---|---|---|
| Most apps (bulk / flat files) | `nfs-storage` | Default class; survives the pod rescheduling to another node. |
| SQLite apps | `local-path` | Node-local, from the **standalone** local-path provisioner (k3s's built-in is disabled). **Required** for anything on SQLite — NFS doesn't do POSIX file locking reliably and *will corrupt the DB*. Pair with `strategy: Recreate` (the volume mounts on one node only). |
| Relational DB (Postgres/MariaDB) | — | Not a PVC — the app connects to the shared database server on the NAS. See the architecture doc. |

!!! warning "`local-path` is a standalone provisioner, not k3s's built-in"
    k3s runs with `--disable local-storage`, so the built-in `local-path` class doesn't exist; a separate standalone provisioner supplies this class instead — **live** since 2026-06. Its StorageClass **must** set `defaultVolumeType: local`, or velero silently backs up nothing — see [Storage & Data Architecture](storage-architecture.md#the-local-path-tier).

## Step 4 — Routing (DNS + HTTPRoute)

**First, publish the hostname.** Service names are individual A records — there is no wildcard record: add the subdomain to `var.services` in the Cloudflare Terraform module (Runbook 8) and apply; each entry becomes one A record pointing at the Traefik LB IP. A browser "server not found" on a freshly deployed app is almost always this step missing — the route below can be perfectly healthy and still unreachable by name. If a record was ever hand-created in the dashboard, `tofu import` it into state instead of letting apply mint a duplicate A record.

Then attach an `HTTPRoute` to the shared Traefik `Gateway`'s `websecure` listener. **The app configures no TLS** — the Gateway terminates it with the cert-manager `*.yourdomain.com` wildcard, so a route is all you need:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <app>
  namespace: <app>
spec:
  parentRefs:
    - name: traefik
      namespace: traefik
      sectionName: websecure
  hostnames:
    - <app>.yourdomain.com
  rules:
    - backendRefs:
        - name: <app>
          port: <svc-port>
```

This replaces the old per-app `IngressRoute` + `certResolver` flow. The backend `Service` lives in the same namespace as the route, so no `ReferenceGrant` is needed.

## Step 5 — Authentication

Pick one mode per app.

### OIDC client (the app speaks OIDC to Authelia)

For apps with their own user system (Immich, Grafana, Forgejo, Vikunja, …). The HTTPRoute stays plain — no middleware. Register the client in `homelab-manifests/apps/authelia/values.yaml` under `configMap.identity_providers.oidc.clients`, commit, and roll Authelia **before** you configure the app:

```yaml
- client_id: 'appname'
  client_name: 'App Display Name'
  client_secret: '<PBKDF2_HASH>'   # gitleaks:allow — inline hash, not a secret
  public: false
  authorization_policy: 'two_factor'
  redirect_uris:
    - 'https://appname.yourdomain.com/<callback-path>'   # app-specific — some need a trailing slash
  scopes: ['openid', 'profile', 'email']
  response_types: ['code']
  grant_types: ['authorization_code']
  token_endpoint_auth_method: 'client_secret_post'   # or client_secret_basic — per app
```

`client_secret` is the **pbkdf2 hash inline** — a plain string, *not* a `value:` sub-map. Generate it by exec'ing into the running pod: `kubectl exec -n authelia authelia-0 -- authelia crypto hash generate pbkdf2 --variant sha512 --password '<secret>'`. The hash is irreversible, so committing it is safe (tag the line `# gitleaks:allow`).

!!! note "Where the *plaintext* secret goes — two cases"
    - **App reads it from its own config** (Immich, Grafana, Forgejo, …): paste the plaintext straight into the app's OAuth settings. **No SealedSecret in the repo** — only the hash (in Authelia's values) and Vaultwarden hold it.
    - **App reads it from a Kubernetes Secret** (Argo CD references `$argocd-oidc:oidc.clientSecret`): seal the plaintext into a `<app>-oidc` SealedSecret labelled `app.kubernetes.io/part-of: <app>`, then reference it from the app's config.

!!! warning "Group → role mapping keys on the lldap group *cn*"
    Authelia forwards each lldap group's **cn verbatim** in the `groups` claim. App RBAC must match that exact name — this cluster uses a dedicated **`homelab-admins`** group (created in lldap), **not** lldap's built-in `lldap_admin`. Membership is captured **at login**, so after changing a user's groups they must fully log out and back in. Apps that read groups from the **userinfo endpoint** (e.g. Argo CD's `enableUserInfoGroups`) get them from the `groups` scope alone; apps that read groups from the **ID token** also need `claims_policy: 'default'` on the client.

### ForwardAuth (Authelia gates the app at the proxy)

For apps with no SSO of their own, or where you just want a 2FA gate in front (lldap UI, Uptime Kuma, …). Define an Authelia ForwardAuth `Middleware` in the app's namespace and reference it from the HTTPRoute with an `ExtensionRef` filter:

```yaml
# middleware.yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: authelia-forwardauth
  namespace: <app>
spec:
  forwardAuth:
    address: http://authelia.authelia.svc.cluster.local:9091/api/authz/forward-auth
    trustForwardHeader: true
    authResponseHeaders: [Remote-User, Remote-Groups, Remote-Name, Remote-Email]
```

```yaml
# in httproute.yaml, add to the rule:
      filters:
        - type: ExtensionRef
          extensionRef:
            group: traefik.io
            kind: Middleware
            name: authelia-forwardauth
```

The Middleware lives in the **app's own namespace** — an `ExtensionRef` filter can't cross namespaces — but its `address` points at the central Authelia service. The route **fails closed**: if Authelia is down, Traefik returns 5xx rather than serving the app unauthenticated. `status.yourdomain.com` (Uptime Kuma) is gated by the catch-all `*.yourdomain.com → two_factor` policy, so no extra `access_control` rule is needed unless you want to relax it (add a `one_factor`/`bypass` rule *above* the catch-all).

!!! note "Verified live (lldap UI + Uptime Kuma)"
    This `ExtensionRef → Middleware` wiring is deployed and working. Two things that make it work, worth re-checking on a major Traefik/Authelia bump:

    1. Traefik's **Kubernetes CRD provider** must be enabled (it is, in this cluster) for `ExtensionRef` middlewares to resolve — moving to the Gateway API does *not* remove that requirement.
    2. The `forward-auth` address and `authResponseHeaders` are Authelia-version-specific — the values above match Authelia 4.39. Confirm against [Authelia's Traefik integration docs](https://www.authelia.com/integration/proxies/traefik/) on a major bump.

!!! tip "Apps with their own login double-prompt"
    If the app has its own auth (Uptime Kuma), you'll authenticate twice — Authelia, then the app. After confirming ForwardAuth works, disable the app's built-in login (Uptime Kuma: **Settings → Security → Disable Auth**). Safe because the route fails closed.

## Step 6 — Register the Application and sync

Add the `Application`(s) to `homelab-manifests/bootstrap/` — the chart app via `$values` plus the `-manifests` app, mirroring `vaultwarden.yaml`. Commit via a branch + PR; on merge the live `root.yaml` app-of-apps creates the Application on its next sync — no manual `kubectl apply`. For stateful apps set `prune: false` on the chart app so a sync can't drop the StatefulSet/PVC out from under it.

## Verification

- [ ] `kubectl get pods -n <app>` — workload Running
- [ ] ArgoCD shows `<app>` and `<app>-manifests` Synced / Healthy
- [ ] `https://<app>.yourdomain.com` serves over the wildcard cert with no TLS warning
- [ ] Auth works — OIDC login round-trips through Authelia, **or** ForwardAuth challenges before the app loads
- [ ] Plaintext secrets saved to Vaultwarden
