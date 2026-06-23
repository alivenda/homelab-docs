# Vikunja

!!! success "Status — Live"
    Live in the cluster, database on NAS PostgreSQL.

Task management — lists, kanban, labels, due dates, reminders, with mobile and
desktop clients.

| | |
|---|---|
| **Difficulty** | Beginner–Intermediate |
| **Time Estimate** | ~1 hour |
| **Runs On** | k3s (app) **+ NAS (database)** |
| **Depends On** | [Deploying an App](apps-deploy-pattern.md), NAS PostgreSQL, Authelia, [Storage & Data Architecture](storage-architecture.md) |

The deployed truth is `homelab-manifests/apps/vikunja/`; this runbook records the
decisions and the bring-up procedure, not the YAML.

## Why this graduated from the App Catalog

Vikunja used to be a catalog row that said `nfs-storage` for everything. That row was
written assuming the default engine — and Vikunja's default is **SQLite**, which makes
"storage: nfs-storage" a quiet instance of the architecture's hard no:
[SQLite-on-NFS corrupts](storage-architecture.md). Vikunja speaks PostgreSQL natively,
so the [decomposition rule](storage-architecture.md#how-to-decompose-one-app) applies
and the relational tier wins:

- **Database** → the shared NAS Postgres (`10.0.20.50:5433`), db + role `vikunja`,
  provisioned per NAS PostgreSQL at bring-up. The nightly dump script enumerates databases
  dynamically, so `vikunja` joins the Garage backup on its first 04:30 run.
- **Attachments / uploaded files** → one `nfs-storage` PVC mounted at
  `/app/vikunja/files` — flat files, NFS-safe. RWO + `strategy: Recreate`.
- **Secrets** → a `SealedSecret` (JWT secret, DB password, OIDC client secret).

A database means a runbook (the catalog's own rule), so the row moved here.

Raw manifests are a *choice* for Vikunja: a first-party Helm chart exists, but its
value-add is bundled Redis/Typesense subcharts this deployment doesn't run. What
actually deploys is one container, one ConfigMap, one PVC.

## Step 1 — Database on the NAS

Per NAS PostgreSQL, on the NAS:

```bash
sudo docker exec -ti postgres psql -U postgres \
  -c "CREATE ROLE vikunja LOGIN PASSWORD '<generated>'" \
  -c "CREATE DATABASE vikunja OWNER vikunja"
```

Append the four per-node `pg_hba.conf` lines scoped to `vikunja vikunja`, then
`SELECT pg_reload_conf()`. Password to the password manager.

Vikunja has **no `database.port` option** — `database.host` carries `host:port` and the
code splits on the last colon (`pkg/db/db.go`, `parsePostgreSQLHostPort`, verified at
v2.3.0). Hence `VIKUNJA_DATABASE_HOST: "10.0.20.50:5433"`.

## Step 2 — The config-surface gotchas

Two things the env-var convention hides:

1. **OIDC providers cannot be configured purely via env.** Vikunja only reads
   `VIKUNJA_AUTH_OPENID_PROVIDERS_<KEY>_*` for provider keys that already exist in a
   config file. The non-secret provider block (issuer, client id, scope) therefore
   lives in a ConfigMap mounted at `/app/vikunja/config.yml`; only the client secret
   arrives via env from the SealedSecret. The mount is `subPath`, so config changes
   need a pod restart.
2. **`enableServiceLinks: false` is mandatory.** The Service is named `vikunja`, so the
   kubelet's discovery env is `VIKUNJA_SERVICE_HOST`, `VIKUNJA_PORT=tcp://…` — squarely
   inside Vikunja's own `VIKUNJA_*` config namespace. Same failure class as the
   `PAPERLESS_PORT` collision in Paperless-ngx.

Also pinned deliberately: `VIKUNJA_SERVICE_JWTSECRET`. Left unset, Vikunja mints a
fresh secret on every start and all sessions die on restart.

## Step 3 — Secrets

Generate the JWT secret and the OIDC client secret (`openssl rand -base64 48` each),
reuse the DB password from Step 1, and seal all three (exact command in the app
README): `VIKUNJA_SERVICE_JWTSECRET`, `VIKUNJA_DATABASE_PASSWORD`,
`VIKUNJA_AUTH_OPENID_PROVIDERS_AUTHELIA_CLIENTSECRET`. Plaintexts to the password
manager.

## Step 4 — SSO: OIDC client

The Authelia client follows the Authelia recipe: `client_id: vikunja`, the **pbkdf2
hash** of the client secret in `apps/authelia/values.yaml`, no PKCE,
`client_secret_basic`, scopes `openid profile email`, and the callback

```
https://tasks.yourdomain.com/auth/openid/authelia
```

— **no trailing slash**, and the last segment is the *provider key* from the
ConfigMap, not the client name. `authurl` on the Vikunja side is the bare Authelia
origin (`https://auth.yourdomain.com`, no path): Vikunja runs OIDC discovery against
it.

No group→team mapping: Vikunja reads teams from a custom `vikunja_groups` claim with a
`{name, oidcID}` structure that Authelia would have to fabricate via a claims policy.
Teams are managed in-app instead. Local registration is off
(`VIKUNJA_SERVICE_ENABLEREGISTRATION=false`); OIDC auto-provisioning is a separate
code path and unaffected — any Authelia user gets a Vikunja account on first login.

## Step 5 — DNS, routing, Application

- `tasks` A record via the Terraform Cloudflare module (`var.services`), `tofu apply`.
- `HTTPRoute` on the shared Gateway, **no ForwardAuth** — Vikunja is an OIDC client
  with its own sessions, API tokens, and mobile/desktop apps; a proxy gate breaks
  every non-browser client.
- One Argo CD `Application` (`bootstrap/vikunja.yaml`), client-side apply — no
  `ServerSideApply` on apps that own an `HTTPRoute`.

## Verification

- [ ] Argo CD: `vikunja` **Synced/Healthy**.
- [ ] `https://tasks.yourdomain.com` loads; version visible at `/api/v1/info`.
- [ ] OIDC round-trip for **two different users** — the second proves
      auto-provisioning, not just your own pre-existing session.
- [ ] Create a task, complete it, reload — state survives (DB write path).
- [ ] Attach a file to a task (files PVC write path).
- [ ] Backup gates: velero `PodVolumeBackup` **bytes** for `vikunja-files` (never
      trust `Completed`), and a `vikunja-<date>.dump` object in the Garage
      `postgres-backups` bucket after the nightly NAS run.
