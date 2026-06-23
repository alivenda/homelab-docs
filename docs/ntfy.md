# ntfy

!!! success "Status — Live"
    Live in the cluster — push notifications for the whole stack.

Self-hosted push notifications for the entire homelab stack.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 45–60 minutes |
| **Runs On** | k3s (cluster) |
| **Depends On** | Kubernetes (k3s), Traefik, Terraform (DNS), [the deploy pattern](apps-deploy-pattern.md) |

ntfy is a pub/sub notification service — services publish messages to named topics and your phone (or any HTTP client) subscribes to receive them. Every other service in this stack can send notifications through it: Prometheus Alertmanager, CI pipeline results, Home Assistant automations.

The ARM64 image (`binwiederhier/ntfy`) ships official multiarch builds. See the [ntfy self-hosting docs](https://docs.ntfy.sh/install/) and [config reference](https://docs.ntfy.sh/config/) for full reference.

## The shape of the deployment

ntfy follows the [deploy pattern](apps-deploy-pattern.md) in **raw-manifests mode**: there is no first-party Helm chart, only third-party repackages — an avoidable dependency for a handful of small files (same call as Homepage). That also means **one** ArgoCD `Application` pointing at `apps/ntfy/manifests/`; with no chart source there is nothing to split a second `-manifests` Application from.

```text
apps/ntfy/manifests/
├── configmap.yaml        # server.yml — including declaratively provisioned users/ACLs
├── deployment.yaml       # pinned tag, strategy: Recreate, node pin
├── pvc.yaml              # ntfy-cache + ntfy-data, both local-path
├── service.yaml          # 80 (http) + 9090 (metrics)
├── httproute.yaml        # plain route — NO ForwardAuth (see below)
├── servicemonitor.yaml   # Prometheus scrape via the metrics listener
└── sealed-tokens.yaml    # SealedSecret: NTFY_AUTH_TOKENS env
```

The workload is a **Deployment with `strategy: Recreate`**, not a StatefulSet: both PVCs are `ReadWriteOnce` on a single node, and explicit PVCs follow the cluster's standard `local-path` pairing (a StatefulSet's `volumeClaimTemplates` also re-introduce an apiserver-defaulting diff that needs Server-Side Diff to silence — avoidable here). **Pin the image tag — never `:latest`.**

## Storage: `local-path` ×2

Both persistent stores are **SQLite**, so both PVCs bind the node-local `local-path` class — *not* `nfs-storage`, which corrupts SQLite (see [Storage & Data Architecture](storage-architecture.md)):

- `ntfy-cache` (2 Gi) → `/var/cache/ntfy` — `cache.db` (message cache) plus the attachment files
- `ntfy-data` (1 Gi) → `/var/lib/ntfy` — `user.db` (users/ACLs/tokens)

Pin the pod to a light worker node with `nodeSelector: kubernetes.io/hostname` — `local-path` provisions the PVs wherever the pod lands, so the pin also fixes where the data lives.

## Configuration (`server.yml` via ConfigMap)

```yaml
base-url: "https://ntfy.yourdomain.com"
listen-http: ":80"
behind-proxy: true            # trust X-Forwarded-For — Traefik fronts this

# iOS only: APNs can't hold a connection to a self-hosted server, so ntfy
# forwards poll requests (message IDs only, never content) to ntfy.sh, which
# fires the Apple push. Omit on an Android-only setup.
upstream-base-url: "https://ntfy.sh"

cache-file: "/var/cache/ntfy/cache.db"
cache-duration: "12h"
attachment-cache-dir: "/var/cache/ntfy/attachments"
attachment-file-size-limit: "15M"
attachment-total-size-limit: "1G"

auth-file: "/var/lib/ntfy/user.db"
auth-default-access: "deny-all"   # private server: all access is explicit
enable-login: true

# Prometheus metrics on a separate listener; the HTTPRoute only routes :80,
# so this stays in-cluster (scraped by the ServiceMonitor).
enable-metrics: true
metrics-listen-http: ":9090"
```

!!! warning "Config changes do not reload"
    ntfy reads `server.yml` (and its env) **once at process start**, and a ConfigMap sync does not roll the pod. After *any* config change — including user, ACL, or token edits below — run `kubectl -n ntfy rollout restart deployment ntfy`.

## Users, ACLs, and tokens — provisioned from git

`auth-default-access: deny-all` makes this a private server: every subscriber and publisher authenticates. Instead of imperative `ntfy user add` (state that lives only in `user.db`), declare everything in config so a rebuilt cluster comes back with its auth intact:

```yaml
# in server.yml — bcrypt hashes are one-way and safe to commit
auth-users:
  - "<YOUR_USERNAME>:$2a$10$...:admin"        # you — the phone subscriber
  - "alertmanager:$2a$10$...:user"            # one account per publishing service
  - "home-assistant:$2a$10$...:user"

# write-only + topic-scoped: a leaked publisher credential can spam its own
# topic but can never read anything
auth-access:
  - "alertmanager:alerts:wo"
  - "home-assistant:home:wo"
```

Generate each hash with the official image (interactive prompt, prints the hash):

```bash
docker run --rm -it binwiederhier/ntfy:<version> user hash
```

For your admin user, hash your real password (store it in your password manager first). The service accounts are **token-only** — hash a random throwaway (`openssl rand -base64 18`) and discard it; the token is their real credential.

**Tokens** must be `tk_` + 29 characters (32 total). They are secrets, so they ride in a `SealedSecret` as the `NTFY_AUTH_TOKENS` env var (env beats `server.yml` in ntfy's precedence) consumed via `envFrom` — never in the ConfigMap:

```bash
kubectl create secret generic ntfy-tokens --namespace ntfy \
  --from-literal=NTFY_AUTH_TOKENS='alertmanager:tk_...,home-assistant:tk_...' \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml > apps/ntfy/manifests/sealed-tokens.yaml
```

Save each token to your password manager — you'll paste them into the publishing services later.

!!! danger "Git owns the provisioned users"
    Provisioned entries are marked as such in `user.db`: **removing a user from `auth-users` deletes it from the database on the next restart**, and CLI edits to provisioned users get overwritten. Manage them in git only. One consequence worth knowing: a clean pod start (1/1 Running, 0 restarts) is itself a check — ntfy refuses to start on malformed provisioning entries.

## DNS, routing, and (no) SSO

1. **Publish the hostname** — add `ntfy` to the service list in the Cloudflare [Terraform](terraform.md) module and apply: one A record per service pointing at the Traefik LB IP. The route can't be reached by name until this exists.
2. **HTTPRoute** per the [deploy pattern](apps-deploy-pattern.md) — plain, on the shared Gateway's `websecure` listener; the wildcard cert terminates at the Gateway.

!!! note "No Authelia in front of ntfy"
    ntfy has **no OIDC support**, and ForwardAuth would break the phone apps and every publishing integration (they authenticate directly against ntfy's own users/tokens). ntfy joins Vaultwarden and Home Assistant on the no-SSO list; its `deny-all` native auth is the gate.

## Verification

- [ ] `kubectl get pods -n ntfy` — Running 1/1, 0 restarts (provisioning parsed)
- [ ] Decode the live token Secret and confirm it holds real `user:tk_...` values, not placeholders:
  `kubectl -n ntfy get secret ntfy-tokens -o jsonpath='{.data.NTFY_AUTH_TOKENS}' | base64 -d`
- [ ] `https://ntfy.yourdomain.com` loads the web UI; your admin login works
- [ ] Anonymous publish rejected: `curl -d test https://ntfy.yourdomain.com/uptime` → 401/403
- [ ] Token publish arrives on the phone:
  `curl -H "Authorization: Bearer tk_..." -d "ntfy is live" https://ntfy.yourdomain.com/uptime`
- [ ] **Backup captured real bytes** — run an on-demand velero backup of the namespace and check `PodVolumeBackup` progress for **non-zero** `totalBytes` on *both* volumes. `Completed` alone proves nothing (see [Backups & DR](backups.md)).

## Mobile app

Install the ntfy [Android](https://play.google.com/store/apps/details?id=io.heckel.ntfy) or [iOS](https://apps.apple.com/app/ntfy/id1625396347) app. Add your server (`https://ntfy.yourdomain.com`), log in as your admin user, and subscribe to the topics (`uptime`, `alerts`, `ci`, `home`). Per-topic subscriptions mean per-channel notification settings on the phone. iOS delivery depends on `upstream-base-url` being set (above).

## Integrating publishers

Each service uses *its own* token against *its own* topic:

- **Prometheus Alertmanager** — configured in kube-prometheus-stack's values, not
  by hand; [Observability Step 4](observability.md#step-4-alerting-ntfy) has the full
  wiring. Two details differ from the other publishers: it uses the **in-cluster**
  URL (`http://ntfy.ntfy.svc.cluster.local/alerts?template=alertmanager`) so alert
  delivery survives an ingress/DNS outage, and the token rides in a SealedSecret
  read via `credentials_file` rather than inline config.

- **Home Assistant** (RESTful notify):

    ```yaml
    notify:
      - name: ntfy
        platform: rest
        resource: https://ntfy.yourdomain.com/home
        method: POST_JSON
        headers:
          Authorization: Bearer <HOME_ASSISTANT_TOKEN>
        data:
          message: "{{ message }}"
    ```

- **CI**: publish from a pipeline step with a plain
  `curl -H "Authorization: Bearer $NTFY_TOKEN" -d "build failed" https://ntfy.yourdomain.com/ci`
  (token injected as a CI secret, never committed).

Adding a publisher later = one `auth-users` hash + one `auth-access` line + re-sealing the token Secret with the extra entry, then a rollout restart.
