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

| Use | StorageClass | Notes |
|---|---|---|
| Most apps | `nfs-storage` | Survives the pod rescheduling to another node. |
| SQLite apps | `local-path` | k3s built-in, node-local. **Required** for anything on SQLite — NFS doesn't do POSIX file locking reliably and *will corrupt the DB*. Pair with `strategy: Recreate` (a local-path volume mounts on one node only). |

!!! warning "`nfs-storage` isn't live yet"
    The `nfs-storage` StorageClass depends on the topaz SSD + NFS export, which is not up yet. Until it is, only `local-path` apps can run. See [Backups & DR](10-backups.md).

## Step 4 — Routing (HTTPRoute)

Apps attach an `HTTPRoute` to the shared Traefik `Gateway`'s `websecure` listener. **The app configures no TLS** — the Gateway terminates it with the cert-manager `*.yourdomain.com` wildcard, so a route is all you need:

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

For apps with their own user system (linkding, Mealie, Vikunja, …). The HTTPRoute stays plain — no middleware. Register the client in `homelab-manifests/apps/authelia/values.yaml` under `identity_providers.oidc.clients`, commit, and upgrade Authelia **before** deploying the app:

```yaml
- client_id: <app>
  client_name: <App>
  client_secret:
    value: <HASHED_SECRET>   # authelia crypto hash generate pbkdf2 --variant sha512
  redirect_uris:
    - https://<app>.yourdomain.com/<callback-path>
  scopes: [openid, profile, email]
  token_endpoint_auth_method: client_secret_basic
```

The callback path is app-specific (the catalog row notes it) — some require a trailing slash.

### ForwardAuth (Authelia gates the app at the proxy)

For apps with no SSO of their own (Uptime Kuma, …). Define an Authelia ForwardAuth `Middleware` in the app's namespace and reference it from the HTTPRoute with an `ExtensionRef` filter:

```yaml
# middleware.yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: authelia-forwardauth
  namespace: <app>
spec:
  forwardAuth:
    address: http://authelia.authelia.svc.cluster.local/api/authz/forward-auth
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

!!! warning "Verified mechanism — two things to confirm at first deploy"
    The `ExtensionRef → Middleware` mechanism is per [Traefik's official Gateway API reference](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/gateway-api/). Two caveats:

    1. Traefik's **Kubernetes IngressRoute (CRD) provider must be enabled** in Traefik's static config for `ExtensionRef` middlewares to resolve — moving to the Gateway API does *not* remove that requirement.
    2. The exact Authelia `forward-auth` address and required headers are Authelia-version-specific — confirm against [Authelia's Traefik integration docs](https://www.authelia.com/integration/proxies/traefik/) at deploy time. **No ForwardAuth app is deployed on the cluster yet**, so this block is unverified against a live cluster.

## Step 6 — Register the Application and sync

Add the `Application`(s) to `homelab-manifests/bootstrap/` — the chart app via `$values` plus the `-manifests` app, mirroring `vaultwarden.yaml`. Commit via a branch + PR and let ArgoCD sync. For stateful apps set `prune: false` on the chart app so a sync can't drop the StatefulSet/PVC out from under it.

## Verification

- [ ] `kubectl get pods -n <app>` — workload Running
- [ ] ArgoCD shows `<app>` and `<app>-manifests` Synced / Healthy
- [ ] `https://<app>.yourdomain.com` serves over the wildcard cert with no TLS warning
- [ ] Auth works — OIDC login round-trips through Authelia, **or** ForwardAuth challenges before the app loads
- [ ] Plaintext secrets saved to Vaultwarden
