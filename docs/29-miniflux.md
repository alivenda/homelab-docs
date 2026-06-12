# Runbook 29: Miniflux

Minimalist RSS/Atom feed reader — a single Go binary with Fever and Google
Reader API endpoints for mobile clients, and very low RAM.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | ~45 minutes |
| **Runs On** | k3s (app) **+ NAS (database)** |
| **Depends On** | [Deploying an App](apps-deploy-pattern.md), Runbook 27 (NAS Postgres), Runbook 18 (Authelia), [Storage & Data Architecture](storage-architecture.md) |

The deployed truth is `homelab-manifests/apps/miniflux/`; this runbook records
the decisions and the bring-up procedure, not the YAML.

## Why this graduated from the App Catalog

The catalog row for this slot was **FreshRSS** — written while the choice was
still open ("pick Miniflux if you prefer lightweight and fast"). Deployment
picked Miniflux, and Miniflux is **PostgreSQL-only**: no SQLite fallback, no
data directory. That changes the shape twice over:

- **A database means a runbook** (the catalog's own rule): db + role
  `miniflux` on the shared NAS Postgres (`10.0.20.50:5433`) per Runbook 27,
  joining the nightly dump → Garage backup automatically.
- **There is no storage tier at all.** The FreshRSS row said `nfs-storage`,
  2 Gi; Miniflux mounts nothing — no PVC, not even an emptyDir. The pod is
  the cluster's first fully stateless workload, which *inverts* the usual
  velero gate (see Verification).

No first-party Helm chart exists — only third-party repackages, the same
avoidable dependency ntfy and Homepage reject — so raw manifests. And because
Miniflux is configured **entirely through environment variables**, there
isn't even a ConfigMap: non-secret env sits inline in the Deployment, secrets
in one SealedSecret. It is the smallest app directory in the repo.

## Step 1 — Database on the NAS

Per Runbook 27, on the NAS — but set the password interactively with
`\password`, not inline in `CREATE ROLE`:

```bash
sudo docker exec -ti postgres psql -U postgres
```

```
CREATE ROLE miniflux LOGIN;
\password miniflux
CREATE DATABASE miniflux OWNER miniflux;
\q
```

!!! warning "Use `\password`, not `CREATE ROLE … PASSWORD '…'`"
    An inline password travels through two shells (yours, then `docker exec`)
    before psql ever sees it. During an earlier bring-up that path silently
    produced a role that rejected the very password just pasted — quoting en
    route is the prime suspect. `\password` prompts inside psql itself,
    bypassing every shell layer, and confirms by double entry.

Generate the password first — `openssl rand -hex 24`, **hex, not base64**:
the value rides inside a connection string, and hex needs no escaping
anywhere. Store it in your password manager, then paste it at the
`\password` prompt.

Append the four per-node `pg_hba.conf` lines scoped to `miniflux miniflux`,
then `SELECT pg_reload_conf()` — Runbook 27 has the exact lines.

## Step 2 — Configuration: env-only, no ConfigMap

Everything non-secret sits inline in the Deployment:

- `LISTEN_ADDR=0.0.0.0:8080` — the default is `127.0.0.1:8080`, which
  neither the Service nor the probes can reach.
- `BASE_URL=https://rss.yourdomain.com` — cookie and redirect base.
- `RUN_MIGRATIONS=1` — migrations run at startup against the NAS; the
  startupProbe allows five minutes for the first boot.
- `CREATE_ADMIN=1` + `ADMIN_USERNAME` (password via the SealedSecret) —
  idempotent: creates the local admin only if absent. This is the bootstrap
  and break-glass account; Step 4 retires it from daily use.
- `METRICS_COLLECTOR=1` + `METRICS_ALLOWED_NETWORKS=10.42.0.0/16` —
  Prometheus scrapes `/metrics` on the app listener, gated by source network
  (scrapes arrive with the Prometheus pod's IP from the pod CIDR), so no
  scrape credentials. A `ServiceMonitor` carrying the
  `release: kube-prometheus-stack` label completes the path (Runbook 9).
- Probes: `GET /healthcheck` — a dedicated route, exempt from
  authentication (verified in `internal/http/server/routes.go` at v2.3.1).

Two gotchas the env-only surface hides:

1. **`DATABASE_URL` accepts the key=value DSN form** — prefer it over the
   URL form: `user=miniflux password=… host=10.0.20.50 port=5433
   dbname=miniflux sslmode=disable`. A URL-form DSN requires
   percent-encoding the password, which is exactly the class of silent
   breakage Step 1's hex password already side-steps.
2. **`enableServiceLinks: false` stays on by convention.** Miniflux reads
   the bare `PORT` variable (a PaaS convention that overrides
   `LISTEN_ADDR`); the kubelet's discovery env for a Service named
   `miniflux` is `MINIFLUX_PORT=tcp://…`, so nothing collides **today** —
   but the `PAPERLESS_PORT`/`VIKUNJA_PORT` class of bug is one Service
   rename away. Off, like everywhere else.

## Step 3 — Secrets

One SealedSecret, three keys (exact seal command in the app README):

| Key | Content |
|---|---|
| `DATABASE_URL` | key=value DSN carrying the Step 1 password |
| `OAUTH2_CLIENT_SECRET` | OIDC client-secret plaintext (`openssl rand -hex 32`) |
| `ADMIN_PASSWORD` | local break-glass admin password |

Plaintexts to your password manager.

## Step 4 — SSO: OIDC client

The Authelia client follows the Runbook 18 recipe — `client_id: miniflux`,
pbkdf2 hash in `apps/authelia/values.yaml` — with one deviation from the
Vikunja client: **`require_pkce: true` / `pkce_challenge_method: 'S256'`**.
Miniflux always sends a PKCE challenge on the authorization request and the
verifier on the exchange (`internal/oauth2/authorization.go`, verified at
v2.3.1), so the client requires what the app already does. Scopes are
exactly what Miniflux requests: `openid profile email` (no `groups` —
Miniflux has no group/role mapping).

The callback is **fixed by Miniflux**, not configurable:

```
https://rss.yourdomain.com/oauth2/oidc/callback
```

— **no trailing slash**. (The superseded FreshRSS catalog row *required* a
trailing slash — do not carry that over.) `OAUTH2_REDIRECT_URL` in the
Deployment and the Authelia `redirect_uris` entry must match this string
verbatim.

On the Miniflux side, `OAUTH2_OIDC_DISCOVERY_ENDPOINT` is the bare Authelia
origin (`https://auth.yourdomain.com`, no path) — Miniflux appends
`/.well-known/openid-configuration` itself.

Account mapping: Miniflux reads the **userinfo endpoint** and takes the
first non-empty of `preferred_username → email → name`. Authelia forwards
the lldap username as `preferred_username`, so accounts land under the
lldap names. `OAUTH2_USER_CREATION=1` auto-provisions any Authelia user on
first OIDC login.

**After** the OIDC round-trip is verified for all users: log in locally as
the admin, promote your own OIDC-provisioned account to administrator
(Settings → Users), then set `DISABLE_LOCAL_AUTH=1` in the Deployment. The
username/password form disappears and OIDC becomes the only door; the
break-glass path is flipping the variable back off via Git.

## Step 5 — DNS, routing, Application

- `rss` A record via the Terraform Cloudflare module (`var.services`), and
  run `tofu apply` **before the first browser lookup** — the UDM caches an
  NXDOMAIN for 1800 s, and one premature lookup costs half an hour of "DNS
  is broken" (learned during the Vikunja bring-up).
- `HTTPRoute` on the shared Gateway, **no ForwardAuth** — Miniflux is an
  OIDC client with its own sessions plus Fever / Google Reader API
  endpoints for mobile readers; a proxy gate breaks every non-browser
  client.
- One Argo CD `Application` (`bootstrap/miniflux.yaml`), client-side apply
  (no `ServerSideApply` on apps that own an `HTTPRoute`) — and, unlike the
  stateful apps, **`prune: true`**: with zero in-cluster state, an
  accidental manifest drop costs one re-sync, not data.

## Verification

- [ ] Argo CD: `miniflux` **Synced/Healthy**.
- [ ] `https://rss.yourdomain.com` loads.
- [ ] OIDC round-trip for **two different users** — the second proves
      auto-provisioning, not just your own pre-existing session.
- [ ] Subscribe to a real feed; entries fetch and render (a live fetch, not
      a spinner).
- [ ] Pod logs clean — no errors, nothing `forbidden`.
- [ ] Homepage tile renders and lands on `https://rss.yourdomain.com`.
- [ ] Prometheus target up (`/metrics` via the ServiceMonitor).
- [ ] **Backup gate (database):** a `miniflux-<date>.dump` object in the
      Garage `postgres-backups` bucket after the nightly NAS run.
- [ ] **Backup gate (inverted):** after the next velero nightly, **zero**
      PVC-backed `PodVolumeBackup` with real bytes in namespace `miniflux`
      — for a stateless pod, a backup showing up means a PVC snuck in.
