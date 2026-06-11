# Runbook 13: Nextcloud

Self-hosted file sync, calendar, contacts — and later Notes, Bookmarks, and Collabora
office editing on top of the same install. Architecturally it's the poster child of the
[Storage & Data Architecture](storage-architecture.md#how-to-decompose-one-app): one app
that touches **every** storage tier, decomposed so each part lands on the right one.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 1–2 hours |
| **Runs On** | k3s — any node. Nothing pins it: files are on NFS, the DB is on the NAS, so the pod can reschedule freely |
| **Depends On** | Runbook 6 (Traefik Gateway), Runbook 18 (Authelia), Runbook 27 (NAS Postgres), and the [GitOps deploy pattern](apps-deploy-pattern.md) |

This runbook follows the [deploy pattern](apps-deploy-pattern.md) — everything is
committed to `homelab-manifests` and reconciled by ArgoCD; the manifests are the source
of truth and this page documents the decisions. What's *specific* to Nextcloud: the data
decomposition, a chart that tempts you into three bundled subcharts you don't want, PHP
reverse-proxy config, background jobs, and OIDC.

## The data layout — four parts, four homes

| Part | Where | Why |
|---|---|---|
| PHP/Apache app pod | the cluster | What the CM4s are good at |
| Files (`/var/www/html` PVC) | `nfs-storage`, 100Gi | Bulk flat files; survives rescheduling to any node |
| Database | NAS Postgres, `10.0.20.50:5433` | Sustained fsync'd writes belong on x86 + a real disk ([R27](27-nas-postgres.md)) |
| Cache + file locking | ephemeral Valkey pod | Not a system of record — no PVC, nothing to back up |

!!! danger "Leave the chart's bundled databases off"
    The [nextcloud/helm](https://github.com/nextcloud/helm) chart ships `postgresql`,
    `mariadb`, and `redis` Bitnami subcharts, and its default is an **internal SQLite
    database** (`internalDatabase.enabled: true`) — on an NFS PVC, that's the
    [SQLite-on-NFS corruption pattern](storage-architecture.md#why-this-isnt-just-use-the-default-storageclass).
    The subcharts now also pull from `bitnamilegacy/*` — the frozen, unmaintained image
    catalog Bitnami left behind in 2025 (the chart even sets
    `global.security.allowInsecureImages: true` to permit them). The values file
    disables all of them: `internalDatabase.enabled: false`, external DB and external
    Redis only.

## Step 1: Provision the database on the NAS

Follow [R27's per-app procedure](27-nas-postgres.md#provisioning-a-database-for-an-app)
on the NAS — generate a password (`openssl rand -base64 24` → password manager first):

```bash
sudo docker exec -ti postgres psql -U postgres \
  -c "CREATE ROLE nextcloud LOGIN PASSWORD '<generated>'" \
  -c "CREATE DATABASE nextcloud OWNER nextcloud"
```

Append the four node-IP lines to `pg_hba.conf` (scoped to exactly this database and
role), then `SELECT pg_reload_conf()` — both verbatim from R27.

The nightly backup needs **no** configuration: the dump script enumerates
`pg_database`, so `nextcloud-<date>.dump` appears in the `postgres-backups` bucket
after the next 04:30 run. Checking that it actually did is part of
[verification](#verification).

## Step 2: One SealedSecret for admin + DB credentials

The chart reads the admin bootstrap credentials and the DB credentials from existing
Secrets — both point at one `SealedSecret`, with key names matching the chart defaults:

```bash
kubectl create secret generic nextcloud-secrets \
  --namespace nextcloud \
  --from-literal=nextcloud-username=admin \
  --from-literal=nextcloud-password='<generated admin password>' \
  --from-literal=db-username=nextcloud \
  --from-literal=db-password='<the password from Step 1>' \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > apps/nextcloud/manifests/nextcloud-sealed.yaml
```

Save both plaintexts to Vaultwarden. The admin credentials are only *consumed* at first
install — the image's autoconfig creates the admin account and connects the DB from
these env vars, so the web setup wizard never appears.

## Step 3: Values — the decisions that matter

`apps/nextcloud/values.yaml` is the source of truth; these are the blocks to understand
rather than copy blindly.

**External database** — the chart takes a colon-delimited port in `host`:

```yaml
internalDatabase:
  enabled: false

externalDatabase:
  enabled: true
  type: postgresql
  host: "10.0.20.50:5433"     # NAS shared Postgres — 5433, not 5432 (R27)
  database: nextcloud
  existingSecret:
    enabled: true
    secretName: nextcloud-secrets
    usernameKey: db-username
    passwordKey: db-password
```

**Files** — one PVC on `nfs-storage` holding all of `/var/www/html` (code, config,
apps, data). The chart can split the data directory onto a second PVC
(`persistence.nextcloudData`), but both would land on the same class here, so it's one
volume until there's a reason.

**Cache + file locking** — a single-container Valkey pod committed to
`apps/nextcloud/manifests/valkey.yaml` (no PVC, no auth, memory-capped, snapshots off),
wired in via the chart's `externalRedis` block. This gives Nextcloud distributed
caching *and* transactional file locking. Don't use the `redis` subchart for this: it's
`bitnamilegacy` (see above) and provisions durable PVCs for what is, by
[architecture](storage-architecture.md#the-four-storage-tiers), ephemeral data.

**Background jobs as a crond sidecar:**

```yaml
cronjob:
  enabled: true      # type: sidecar (default) — crond next to Apache, same pod
```

The chart's alternative (`type: cronjob`, a real Kubernetes `CronJob`) needs the
app volume mountable by a second pod; the sidecar shares the existing pod's mounts and
is the simpler correct choice for a single-replica app. Step 6 flips Nextcloud to
actually *use* it.

**A startup probe, or the installer gets killed:**

```yaml
startupProbe:
  enabled: true
  failureThreshold: 60   # up to ~10 min of grace
```

!!! warning "First boot does minutes of work — on ARM + NFS, many minutes"
    The first start unpacks the Nextcloud release onto the NFS PVC and runs the
    installer. The chart's liveness probe (10s delay) will kill and restart the pod
    mid-install without a startup probe, looping forever. This replaces the old
    runbook's "pin to a 32 GB node" advice — the eMMC size never mattered; probe
    patience did.

**Reverse-proxy correctness** — TLS terminates at the Traefik Gateway, so Nextcloud
must be told it's behind an HTTPS proxy or it builds `http://` redirect URLs and
distrusts the forwarded headers:

```yaml
nextcloud:
  extraEnv:
    - name: OVERWRITEPROTOCOL
      value: "https"
    - name: TRUSTED_PROXIES
      value: "10.42.0.0/16"
```

!!! note "Pod CIDR here — node IPs for NAS-hosted apps"
    An in-cluster app sees the proxy as the **Traefik pod's IP** (the k3s pod CIDR,
    `10.42.0.0/16`). Home Assistant (R16) needed **node** IPs for the same setting
    because it lives *outside* the cluster, where traffic leaves SNAT'd. Same concept,
    different vantage point — don't copy one into the other.

`nextcloud.trustedDomains` needs nothing: it defaults to `nextcloud.host`.

## Step 4: Routing

The hostname is already in `var.services` in the Cloudflare module (R8) — verify with
`tofu plan` before assuming. The route goes in the `-manifests` app, and the backend
port is the chart's Service default, **8080**:

```yaml
# apps/nextcloud/manifests/httproute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nextcloud
  namespace: nextcloud
spec:
  parentRefs:
    - name: traefik
      namespace: traefik
      sectionName: websecure
  hostnames:
    - nextcloud.yourdomain.com
  rules:
    - backendRefs:
        - name: nextcloud
          port: 8080
```

!!! warning "The chart can render this route itself now — don't let it"
    Chart ≥ 9 ships `httpRoute.enabled`. Tempting, but the chart `Application` syncs
    with Server-Side Apply, and SSA + a Gateway API `HTTPRoute` is the
    [permanent-OutOfSync trap](07-vaultwarden.md#step-3-httproute). The route stays in
    the no-SSA `-manifests` app like every other app's.

No auth middleware on this route — Nextcloud is an OIDC app (Step 7), and ForwardAuth
would break every sync client ([R18](18-authelia.md)).

## Step 5: Register the Applications and sync

`bootstrap/nextcloud.yaml` holds the usual pair ([deploy pattern](apps-deploy-pattern.md),
mirroring `vaultwarden.yaml`): the chart app — `nextcloud/helm` pinned with a
`# renovate:` comment, multi-source `$values`, `prune: false` — and `nextcloud-manifests`
(HTTPRoute + Valkey + SealedSecret, no SSA). Branch, PR, merge, and watch the install:

```bash
kubectl -n nextcloud logs deploy/nextcloud -f
kubectl get pods -n nextcloud
```

Expect several quiet minutes (see the startup-probe note) before
`https://nextcloud.yourdomain.com` serves a login page directly — no setup wizard, since
autoconfig already ran with the sealed credentials.

## Step 6: Flip background jobs to cron

The crond sidecar fires `cron.php` every 5 minutes, but Nextcloud still defaults to
AJAX-triggered jobs (runs only while someone has a tab open — unreliable for a sync
server). Flip the mode once, post-install:

```bash
kubectl -n nextcloud exec deploy/nextcloud -c nextcloud -- \
  su -s /bin/sh -c "php occ background:cron" www-data
```

**Settings → Administration → Basic settings** should show *Cron* selected and, within
five minutes, a fresh *Last job execution*.

## Step 7: OIDC login through Authelia

Nextcloud is the textbook [OIDC-mode app](apps-deploy-pattern.md#oidc-client-the-app-speaks-oidc-to-authelia):
it has its own user system, and its desktop/mobile clients authenticate through Login
Flow v2 (a real browser window), which round-trips OIDC fine. The integration uses
**user_oidc**, Nextcloud's first-party OIDC backend app.

Register the client in `apps/authelia/values.yaml` (per the
[Authelia Nextcloud guide](https://www.authelia.com/integration/openid-connect/clients/nextcloud/):
PKCE required, `client_secret_post`), and roll Authelia **before** touching Nextcloud:

```yaml
# Nextcloud (user_oidc). Login only — no groups scope, no admin mapping; the
# sealed local admin stays the admin (and the occ escape hatch). The plaintext
# secret lives in user_oidc's provider config + Vaultwarden; only the hash here.
- client_id: 'nextcloud'
  client_name: 'Nextcloud'
  client_secret: '<PBKDF2_HASH>'   # gitleaks:allow — pbkdf2 hash, not a secret
  public: false
  authorization_policy: 'two_factor'
  require_pkce: true
  pkce_challenge_method: 'S256'
  redirect_uris:
    - 'https://nextcloud.yourdomain.com/apps/user_oidc/code'
  scopes:
    - 'openid'
    - 'profile'
    - 'email'
  response_types:
    - 'code'
  grant_types:
    - 'authorization_code'
  access_token_signed_response_alg: 'none'
  userinfo_signed_response_alg: 'none'
  token_endpoint_auth_method: 'client_secret_post'
```

Then install and configure the app — three `occ` one-liners and one config key:

```bash
NC_EXEC="kubectl -n nextcloud exec deploy/nextcloud -c nextcloud --"

# 1. Nextcloud's HTTP client refuses private-range targets by default, and
#    auth.yourdomain.com resolves to the Traefik LB on the Lab VLAN — without
#    this, provider discovery fails with a misleading "could not fetch" error.
$NC_EXEC su -s /bin/sh -c "php occ config:system:set allow_local_remote_servers --value=true --type=boolean" www-data

# 2. Authelia requires client_secret_post for this client; user_oidc defaults to basic.
$NC_EXEC su -s /bin/sh -c "php occ config:system:set user_oidc default_token_endpoint_auth_method --value=client_secret_post" www-data

# 3. Install the app and register the provider. unique-uid=0 + the uid mapping
#    keep Nextcloud usernames human (matching lldap) instead of opaque hashes —
#    file ownership is tied to the uid, so get this right BEFORE first OIDC login.
$NC_EXEC su -s /bin/sh -c "php occ app:install user_oidc" www-data
$NC_EXEC su -s /bin/sh -c "php occ user_oidc:provider Authelia \
  --clientid=nextcloud \
  --clientsecret='<plaintext secret>' \
  --discoveryuri=https://auth.yourdomain.com/.well-known/openid-configuration \
  --unique-uid=0 \
  --mapping-uid=preferred_username" www-data
```

The login page now offers **Login with Authelia** alongside the local form. The local
form stays — it's the admin account's only way in, and the break-glass path if Authelia
is down (`/login?direct=1` if the OIDC redirect is ever made automatic).

## Step 8: Prove the backups

Two backup paths, two gates — and per the house rule, gate on **bytes and objects**,
never on `Completed`:

**Files (velero):**

```bash
velero backup create nextcloud-bytes-check \
  --include-namespaces nextcloud \
  --default-volumes-to-fs-backup \
  --wait

kubectl -n velero get podvolumebackup \
  -l velero.io/backup-name=nextcloud-bytes-check \
  -o custom-columns='VOLUME:.spec.volume,PHASE:.status.phase,TOTAL:.status.progress.totalBytes,DONE:.status.progress.bytesDone'
```

The `nextcloud-main` volume must show `TOTAL`/`DONE` clearly non-zero — a fresh install
is hundreds of MB. (The Valkey pod contributes nothing durable; an emptyDir row is
noise, not signal.)

**Database (R27 nightly dump)** — after the next 04:30 run, on the NAS:

```bash
sudo docker exec -ti garage /garage bucket info postgres-backups
```

Object count up by one, with a `nextcloud-<date>.dump` of non-trivial size.

## Verification

- [ ] `kubectl get pods -n nextcloud` — app pod `2/2` Running (Apache + crond sidecar), Valkey Running.
- [ ] ArgoCD shows `nextcloud` and `nextcloud-manifests` Synced / Healthy.
- [ ] `https://nextcloud.yourdomain.com` loads the login page over the wildcard cert — no setup wizard.
- [ ] **Login with Authelia** round-trips: 2FA at Authelia, lands back in Nextcloud as your lldap user (human-readable username, not a hash).
- [ ] Desktop client syncs a file; it appears on a second device/web within a minute.
- [ ] Admin → Basic settings: mode is *Cron*, *Last job execution* under 5 minutes old.
- [ ] Admin → Overview security check shows no reverse-proxy or `overwriteprotocol` warnings.
- [ ] Velero `PodVolumeBackup` for the Nextcloud volume shows **bytes > 0** (Step 8).
- [ ] `nextcloud-<date>.dump` landed in `postgres-backups` after the nightly run.
- [ ] Admin password, DB password, and OIDC client secret all in Vaultwarden.

!!! tip "Later"
    The chart bundles a serverinfo Prometheus exporter (`metrics.enabled`) and a
    Collabora subchart — both deliberately out of scope here. Collabora gets its own
    [App Catalog entry](apps-catalog.md#collabora-online); wire metrics into
    kube-prometheus-stack when there's a dashboard to justify the token setup.
