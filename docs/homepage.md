# Homepage

!!! success "Status — Live"
    Live in the cluster behind Authelia ForwardAuth.

A services dashboard — one tile per deployed service, behind Authelia.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 45–60 minutes |
| **Runs On** | k3s (cluster) |
| **Depends On** | Traefik, Terraform (DNS), Authelia (ForwardAuth), [the deploy pattern](apps-deploy-pattern.md) |

Homepage ([gethomepage.dev](https://gethomepage.dev)) is a YAML-configured start page. In this stack it serves static tiles and bookmarks for every live service, reads the Kubernetes API for cluster stats, and sits entirely behind Authelia ForwardAuth.

The `ghcr.io/gethomepage/homepage` image ships official multiarch builds — **pin a release tag and confirm `arm64` is on the manifest when you bump** (check the ghcr manifest list, not just the release notes). The deployed pin at time of writing is `v1.13.2`.

!!! note "ForwardAuth — not OIDC"
    Homepage has no login page and no OIDC support. It is protected by the Authelia ForwardAuth middleware (Authelia, same recipe as lldap). Do **not** attempt to register it as an OIDC client.

## The shape of the deployment

Homepage follows the [deploy pattern](apps-deploy-pattern.md) in **raw-manifests mode**: there is no first-party Helm chart — upstream's own docs link an unofficial one, the same avoidable third-party dependency ntfy rejects. One ArgoCD `Application` pointing at `apps/homepage/manifests/`:

```text
apps/homepage/manifests/
├── rbac.yaml         # ServiceAccount + ClusterRole + binding (read-only, narrowed)
├── configmap.yaml    # the entire dashboard — all nine config files
├── deployment.yaml   # pinned tag, stateless, no node pin
├── service.yaml      # 3000 (http)
├── middleware.yaml   # Authelia ForwardAuth (namespace-local copy)
└── httproute.yaml    # ExtensionRef → the middleware
```

## Stateless on purpose

Homepage's configuration is YAML — it belongs in git, not on a volume. There is **no PVC**:

- All nine config files ride in the ConfigMap.
- The only path the image insists on writing (`/app/config/logs`) is an `emptyDir`.
- Widget API tokens (the section below) ride a `SealedSecret` as `HOMEPAGE_VAR_*` env — never values in the ConfigMap.

That buys three things. The Application can run `prune: true` (an accidental manifest drop costs one re-sync, not data). A rebuild is a re-sync — there is nothing to restore. And the velero gate **inverts**: instead of proving bytes were backed up, prove *nothing* is:

```bash
kubectl get podvolumebackups -n velero \
  -o custom-columns='NS:.spec.pod.namespace,VOL:.spec.volume,BYTES:.status.progress.totalBytes' \
  | grep homepage
```

After the first nightly, every `homepage` row must be an emptyDir at `<none>` bytes — velero's opt-out fs-backup sweeps emptyDirs too, so the noise rows are expected (same as Collabora and Traefik). A row with **real bytes** means a PVC crept in and the statelessness claim is broken.

## RBAC — narrowed from upstream's example

Homepage reads the Kubernetes API for two features: the **kubernetes info widget** (cluster/node CPU + memory, served by k3s's bundled metrics-server) and **route discovery**. Upstream's example ClusterRole also grants `ingresses` (extensions/networking.k8s.io) and `ingressroutes` (traefik.io) — this cluster has no such objects, routing is exclusively Gateway API `HTTPRoute`, so those grants are dropped and the matching discovery paths are switched off in `kubernetes.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: homepage
rules:
  - apiGroups: [""]
    resources: ["namespaces", "pods", "nodes"]
    verbs: ["get", "list"]
  - apiGroups: ["gateway.networking.k8s.io"]
    resources: ["httproutes", "gateways"]
    verbs: ["get", "list"]
  - apiGroups: ["metrics.k8s.io"]
    resources: ["nodes", "pods"]
    verbs: ["get", "list"]
```

Plus a `ServiceAccount` (with `automountServiceAccountToken: true` — the in-pod token is the point) and a `ClusterRoleBinding`.

## Discovery: HTTPRoutes are supported — and opt-in

Earlier revisions of this runbook hedged on whether discovery was Ingress-only. Verified against upstream for v1.13.x: **Gateway API HTTPRoute discovery is real**, enabled with `gateway: true`, reading the same `gethomepage.dev/*` annotations as Ingress.

```yaml
# kubernetes.yaml
mode: cluster     # use the in-pod ServiceAccount
gateway: true     # discover annotated HTTPRoutes
ingress: false    # no Ingress objects exist — and RBAC doesn't grant them
traefik: false    # ditto IngressRoutes
```

Discovery is **opt-in per route**: nothing appears until an HTTPRoute carries `gethomepage.dev/enabled: "true"` (plus `name`/`group`/`icon`/`description`). The deployed dashboard annotates no routes — every tile is static in `services.yaml`, one PR, no churn across sixteen app manifests. Annotating routes is the forward path if static upkeep gets old: new apps would then tile themselves.

## Configuration — the ConfigMap is the dashboard

Homepage expects nine files under `/app/config`; **all nine must exist as ConfigMap keys** — `settings.yaml`, `widgets.yaml`, `services.yaml`, `bookmarks.yaml`, `kubernetes.yaml`, `docker.yaml` (`{}` — no Docker socket on k3s), `proxmox.yaml` (`{}` — no Proxmox widgets), `custom.js`, `custom.css` (empty strings) — because the mounted directory is read-only and the app cannot create missing ones.

!!! warning "The required set is the skeleton directory, not the documented list"
    What "expects" means precisely: at startup Homepage copies any file missing from `/app/config` out of its image's `src/skeleton/` directory. On a read-only mount that copy fails with `EROFS` and the process **exits 1** — *after* the web server has already bound, so the pod flashes Ready, drops, and crash-loops. The deployed v1.13.2 skeleton holds nine files (`proxmox.yaml` is the easy one to miss); on every image bump, diff `src/skeleton/` at the new tag against the ConfigMap keys before rolling.

!!! tip "Whole-dir mount, not subPath"
    Upstream's k8s example mounts each file with `subPath`, which **freezes** the file — kubelet never updates subPath mounts, so every config edit needs a pod restart. Mounting the ConfigMap volume whole at `/app/config` (with the `emptyDir` shadowing `logs/` inside it) keeps kubelet's atomic-symlink update path: an ArgoCD sync reaches the pod within about a minute and shows on page refresh. If an edit doesn't appear: `kubectl -n homepage rollout restart deployment homepage`.

`widgets.yaml` ships two **keyless** info widgets — `kubernetes` (cluster + node CPU/memory via the ServiceAccount) and `datetime`. `services.yaml` is the tile grid, grouped to match `settings.yaml`'s `layout` (group names must match exactly):

```yaml
# services.yaml (abridged — one entry per live service)
- Apps:
    - Nextcloud:
        href: https://nextcloud.yourdomain.com
        description: Files, calendar, contacts
        icon: nextcloud.png
    - Vikunja:
        href: https://tasks.yourdomain.com
        description: Tasks & projects
        icon: vikunja.png
    # ... Immich, Paperless-ngx, Vaultwarden, Home Assistant, ntfy

- Operations:
    - ArgoCD:
        href: https://argocd.yourdomain.com
        description: GitOps
        icon: argo-cd.png
    # ... Grafana, Forgejo, Woodpecker

- Infrastructure:
    - Traefik:
        href: https://traefik.yourdomain.com
        description: Ingress gateway
        icon: traefik.png
    # ... Authelia, lldap, Collabora
```

The source of truth for *which* tiles exist is the cloudflare module's `var.services` plus the [App Catalog](apps-catalog.md) — if a subdomain has an A record, it gets a tile. `bookmarks.yaml` holds the external links (GitHub, Cloudflare dash, UniFi).

!!! tip "Order widget-bearing tiles first within each group"
    The `row` layout makes every tile in a row share the tallest tile's height. A widget tile (it carries a stats strip) is taller than a plain tile, so interleaving the two leaves gaps under the plain ones. Put the widget-bearing services at the **top of each group** and the plain tiles after; rows then come out even. Leave a comment saying so — "tidying" the list back into semantic order re-rags the grid.

Icons resolve against the [dashboard-icons](https://github.com/homarr-labs/dashboard-icons) set by bare filename. **Verify each name exists** (a typo renders a broken image, not an error) — the two non-obvious ones in this stack are `argo-cd.png` (not `argocd`) and `lldap.png` (not `ldap`).

## Deployment

The Deployment is where the gotchas live. Pinned image, default `RollingUpdate`, and **no node pin** — with no volume binding it to a node, the scheduler may place and move it freely; this is the stack's first app where that's true.

```yaml
spec:
  template:
    spec:
      serviceAccountName: homepage
      automountServiceAccountToken: true
      enableServiceLinks: false        # see below — load-bearing
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000                  # makes the logs emptyDir writable for 1000
        runAsNonRoot: true
      containers:
        - name: homepage
          image: ghcr.io/gethomepage/homepage:v1.13.2   # pinned — never :latest
          ports:
            - name: http
              containerPort: 3000
          env:
            - name: MY_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: HOMEPAGE_ALLOWED_HOSTS
              value: "homepage.yourdomain.com,$(MY_POD_IP):3000"
          volumeMounts:
            - name: config
              mountPath: /app/config
              readOnly: true
            - name: logs
              mountPath: /app/config/logs
      volumes:
        - name: config
          configMap:
            name: homepage-config
        - name: logs
          emptyDir: {}
```

!!! warning "`enableServiceLinks: false` is load-bearing"
    The Service is named `homepage`, so kubelet's legacy Docker-link env injection would set `HOMEPAGE_SERVICE_HOST`, `HOMEPAGE_PORT`, … — directly inside the `HOMEPAGE_*` env namespace the app itself reads. Third instance of this collision class in the stack (`PAPERLESS_PORT` and Vikunja's `VIKUNJA_*` service links before it). Any app whose env prefix matches its Service name needs this line.

!!! warning "`HOMEPAGE_ALLOWED_HOSTS` must include the pod IP"
    The variable is a mandatory Host-header allow-list (requests with any other Host get rejected). Kubelet probes address the **pod IP**, not the public hostname — omit the `$(MY_POD_IP):3000` entry and the pod fails its own health checks and never goes Ready. `MY_POD_IP` must be declared *before* the line that expands it.

Non-root works cleanly: the image ships every file chowned `1000:1000`, and its entrypoint only needs root for the optional `PUID`/`PGID` re-chown path — unset (the default), it skips straight to `exec`. Probes hit `/api/healthcheck` on 3000.

## HTTPRoute + ForwardAuth

The standard recipe from Authelia, deployed and proven on lldap: the Traefik `Middleware` is copied into the app's own namespace (an `ExtensionRef` filter cannot cross namespaces), pointing at the central Authelia service; the `HTTPRoute` references it as a filter. Fails closed — if Authelia is down the route 5xx's rather than serving the dashboard unauthenticated.

```yaml
rules:
  - filters:
      - type: ExtensionRef
        extensionRef:
          group: traefik.io
          kind: Middleware
          name: authelia-forwardauth
    backendRefs:
      - name: homepage
        port: 3000
```

The ArgoCD `Application` uses client-side apply — **no `ServerSideApply=true`**: the Gateway controller defaults fields on the HTTPRoute after apply, and under SSA the predicted merge never matches the mutated live object, leaving the app permanently OutOfSync. Same as every other HTTPRoute app in the stack.

## Widget API tokens — seven live, the rest tile-only

Homepage can decorate a tile with live per-service stats, each fed by an API token. The dashboard shipped tile-only and grew widgets one at a time — the rule for *whether* a service gets one is **only when a read-only-scopable token buys a stat you'll actually read**. Seven carry a widget today:

| Widget | In-cluster endpoint | Stat it earns |
|--------|--------------------|---------------|
| Nextcloud | *public* URL (see below) | free space, users, files, shares |
| Immich | `immich-nas.immich.svc:2283` | photo / video counts |
| Paperless-ngx | `paperless.paperless.svc:8000` | documents, inbox |
| Forgejo | `forgejo-http.forgejo.svc:3000` | repos, issues, PRs |
| ArgoCD | `argocd-server.argocd.svc:80` | apps synced / out-of-sync / degraded |
| ntfy | `ntfy.ntfy.svc` | unread count on the `alerts` topic |

The other live services stay **tile-only on purpose** — the stat didn't earn a stored, rotatable credential:

- **Grafana** — stateless here, so its UI users are wiped on every restart; the only durable credential is an *admin* service-account token. Storing admin creds to render a panel count fails the value-vs-blast-radius test.
- **Home Assistant** — its long-lived access tokens cannot be scoped read-only; a widget token would be full control of the home.
- **Plex** — the only stat is active-stream count, and streaming is local-only here. No remote value.
- **Vikunja / Miniflux** — task and unread counts that duplicate what the apps surface themselves; not worth a per-app token.

When in doubt, leave the tile plain.

### Why in-cluster URLs (and Nextcloud's exception)

Widget requests originate **server-side**, from the Homepage pod — which has no Authelia session. Pointed at a public `https://…yourdomain.com` URL they'd bounce off ForwardAuth (302 → login). So each widget targets the **in-cluster Service** (`http://<svc>.<ns>.svc.cluster.local:<port>`), reaching the app behind the proxy.

**Nextcloud** is the exception: its `trusted_domains` rejects any Host but the public name, so its widget keeps the public URL — reachable only because Nextcloud's own check, not Authelia, gates that path.

### The token plumbing

Every token rides one `SealedSecret` (`homepage-widget-tokens`, namespace `homepage`) as a `HOMEPAGE_VAR_*` key; the Deployment pulls the lot in with `envFrom`, and the config references `{{HOMEPAGE_VAR_XXX}}` — Homepage substitutes the env at render.

```bash
kubectl create secret generic homepage-widget-tokens \
  --namespace homepage \
  --from-literal=HOMEPAGE_VAR_FORGEJO_TOKEN=<token> \
  --from-literal=HOMEPAGE_VAR_PAPERLESS_TOKEN=<token> \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets --format yaml \
  > apps/homepage/manifests/sealed-widget-tokens.yaml
```

| Service | Where to get the token |
|---------|------------------------|
| Nextcloud | User Settings → Security → App passwords |
| Paperless-ngx | Profile → API Auth Token |
| Forgejo | User Settings → Applications → Generate Token |
| Immich | Account Settings → API Keys |
| ArgoCD | dedicated `readonly` account — see below |
| ntfy | read-only `homepage` publisher — see below |

Rotating any widget token is reseal, merge, restart — in that order. `kubeseal`
re-encrypts *all* keys each run, so the whole file changes even when only one token
changed — have every widget's plaintext on hand, not just the one you're rotating, and
store each in the password manager as you mint it. (The repo's gitleaks pre-commit hook
flags every `encryptedData` blob as a generic API key; clear each with a per-line
fingerprint in `.gitleaksignore` rather than weakening the hook.) Unlike widget *config*
(URLs, fields — a ConfigMap whole-dir mount that lands on the next page refresh, no
restart needed), widget *tokens* are env, read once at process start: after merging a
SealedSecret change you must `kubectl -n homepage rollout restart deployment homepage`.

!!! danger "Don't restart into the propagation race"
    Restarting the consumer *immediately* after merging a SealedSecret change races the sealed-secrets controller: the new pod can start **before** the controller has decrypted and written the plain Secret, so it loads stale creds and the widget 401s. Diagnose by comparing the pod's `startTime` against the Secret's write time (`kubectl get secret … -o jsonpath='{.metadata.managedFields[*].time}'`); the fix is to restart **again** once that write timestamp is in the past — not to reseal. (This bit both the ntfy and homepage rollouts.)

### Two service-specific gotchas

**ArgoCD** has no way to mint a non-expiring API token for `admin` — the built-in `admin` is password-only break-glass by design (asking for one returns `account 'admin' does not have apiKey capability`). Add a dedicated read-only account with the apiKey capability instead:

```yaml
# argocd-cm
accounts.readonly: apiKey
# argocd-rbac-cm  policy.csv
g, readonly, role:readonly
```

then mint against `readonly`: with the CLI, `argocd account generate-token --account readonly`; with no CLI, two REST calls — `POST /api/v1/session` (admin creds) for a session JWT, then `POST /api/v1/account/readonly/token`. **Send the bodies as JSON** — curl's `-d` defaults to form-encoding, ArgoCD answers with a non-JSON error, and the outer `jq` chokes; add `-H 'Content-Type: application/json'`.

**ntfy** widgets read the `alerts` topic, so the dashboard gets its own **read-only** publisher: a `homepage` user (bcrypt hash in the configmap — committable) with `homepage:alerts:ro` access, plus a `tk_…` token in the token SealedSecret. ntfy reads users and tokens only at startup, and provisioning a user doesn't change the Deployment spec — so it won't auto-roll; `kubectl -n ntfy rollout restart deployment ntfy` after merge. (See [ntfy — ntfy](ntfy.md) for the provisioning model.)

## Verification

- [ ] ArgoCD app `Synced/Healthy`; pod Running:

    ```bash
    kubectl get pods -n homepage
    ```

- [ ] `https://homepage.yourdomain.com` redirects unauthenticated requests to Authelia; after login the dashboard renders. Repeat for **both** users.
- [ ] Spot-check tiles — including the newest services — land on the right URLs.
- [ ] Kubernetes widget shows live cluster CPU/memory and all four nodes (data, not spinners).
- [ ] All seven service widgets render live numbers, not an error or a spinner — a 401/timeout means a wrong token, the public-vs-in-cluster URL, or the [propagation race](#widget-api-tokens-seven-live-the-rest-tile-only).
- [ ] No RBAC 403s in the pod log:

    ```bash
    kubectl logs -n homepage deploy/homepage | grep -i forbidden
    ```

- [ ] **Inverted velero gate** after the first nightly: zero PVC-backed `PodVolumeBackup` rows with real bytes in namespace `homepage` (emptyDir rows at `<none>` are expected noise).
