# Runbook 14: Paperless-ngx

Scan, OCR, and organize all your documents.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 1–2 hours |
| **Runs On** | k3s (app + Redis) **+ NAS (database)** |
| **Depends On** | [Deploying an App](apps-deploy-pattern.md), Runbook 27 (NAS Postgres), Runbook 18 (Authelia), [Storage & Data Architecture](storage-architecture.md) |

Paperless-ngx is the cleanest live example of the
[decomposition rule](storage-architecture.md#how-to-decompose-one-app): one app, four
data homes. The deployed truth is `homelab-manifests/apps/paperless/`; this runbook
records the decisions and the bring-up procedure, not the YAML.

## Why raw manifests — there is no official chart

The paperless-ngx project publishes **no first-party Helm chart**. Its GitHub org
contains only `paperless-ngx`, `ansible`, `builder`, and `gh-docs-index` (verified
2026-06), and the upstream-documented deployments are docker compose and bare metal.
Everything on Artifact Hub is a third-party repackage — the same avoidable dependency
this cluster already rejected for ntfy.

So the workload is **raw manifests**: one Deployment (image pinned to the upstream
`ghcr.io/paperless-ngx/paperless-ngx` release), a small Redis Deployment, PVCs, a
Service, an HTTPRoute — one ArgoCD `Application` in `bootstrap/paperless.yaml`, no
`-manifests` split (there's no chart source to split from).

!!! warning "If you read an older revision of this runbook"
    Earlier drafts installed a community chart imperatively (`helm upgrade --install`)
    with **postgres and redis bundled in-cluster**, and pinned the pod via a
    `workload: heavy` node label. All three are wrong here: the chart URL never
    existed, in-cluster postgres contradicts the storage architecture, and no node
    carries that label. GitOps-only, database on the NAS, no node pin.

## The four data homes

| Component | Home | Why |
|---|---|---|
| Relational DB | NAS shared Postgres `10.0.20.50:5433`, db + role `paperless` | The [relational tier](storage-architecture.md#relational-databases-on-the-nas-not-the-cluster) — real databases don't run on CM4 eMMC |
| Documents, search index, consume, export | Four `nfs-storage` PVCs | Bulk flat files. With the DB external there is **no SQLite in the data dir**, so NFS is safe |
| Redis (Celery broker + cache) | In-cluster `paperless-redis`, emptyDir, persistence off | Cache tier — not a system of record, nothing to back up (velero-excluded) |
| Credentials + OIDC provider config | `paperless-secrets` SealedSecret | The OIDC JSON embeds a plaintext secret — see Step 3 |

!!! danger "The consume dir is NFS — polling is not optional"
    inotify events don't propagate across NFS clients, so a consumer relying on
    filesystem notifications **never sees new files** and scan-to-archive silently
    does nothing. `PAPERLESS_CONSUMER_POLLING=60` is set in the Deployment; keep it
    when touching env.

!!! danger "A Service named `paperless` poisons `PAPERLESS_PORT`"
    Kubernetes injects Docker-link-style discovery env
    (`<SERVICE>_PORT=tcp://<ClusterIP>:<port>`) for every Service in the
    namespace — so the app's own Service injects
    `PAPERLESS_PORT=tcp://10.43.x.x:8000`, the exact variable paperless-ngx reads
    as its webserver port. Granian exits on the non-integer value, the pod loops
    on its startup probe, and the route serves Traefik's "no available server"
    503 while everything *around* the pod looks healthy. The Deployment sets
    `enableServiceLinks: false` — keep it. (Caught live at bring-up: init
    completed in 56 s, then the webserver died on its first instruction.)

There is deliberately **no nodeSelector**: documents are on NFS, the database is on
the NAS, the broker is ephemeral — nothing node-local exists to reschedule back to.
Resource requests steer placement; the 2 Gi memory limit absorbs OCR spikes.

## Step 1 — Database on the NAS

Follow [Runbook 27 → Provisioning a database for an app](27-nas-postgres.md#provisioning-a-database-for-an-app)
with `<app> = paperless`: create the role + database, append the four per-node
`pg_hba.conf` lines scoped to exactly this db/role, and `pg_reload_conf()`. The role
password goes to your password manager and into the SealedSecret in Step 2.

The nightly dump script enumerates databases dynamically, so `paperless` joins the
Garage backup on its first 04:30 run — **no script change**, but verify the object
appears (Verification below).

## Step 2 — Secrets

One SealedSecret, four keys, sealed per the
[standard pattern](apps-deploy-pattern.md): `PAPERLESS_SECRET_KEY` (generate, e.g.
`openssl rand -base64 48`), `PAPERLESS_DBPASS` (the Step 1 role password),
`PAPERLESS_ADMIN_PASSWORD` (the local break-glass superuser, auto-created at first
start), and `PAPERLESS_SOCIALACCOUNT_PROVIDERS` (Step 3). The exact seal command is
in `apps/paperless/README.md`.

## Step 3 — SSO: OIDC client, with one twist

Paperless speaks OIDC natively via django-allauth, so it takes the **OIDC client**
mode from [the app pattern](apps-deploy-pattern.md) — the route stays plain (the
mobile app and API tokens authenticate directly; ForwardAuth would break both).

Register the client in Authelia's values (already done in
`apps/authelia/values.yaml`), per the
[Authelia Paperless guide](https://www.authelia.com/integration/openid-connect/clients/paperless/):
PKCE `S256`, `client_secret_basic`, and a callback whose **trailing slash is
load-bearing**:

```text
https://paperless.yourdomain.com/accounts/oidc/authelia/login/callback/
```

The twist: paperless has **no settings UI for OIDC** — it reads the whole provider
config, *including the plaintext client secret*, from one env var. So unlike the
"paste the plaintext into the app's OAuth settings" case, the plaintext here lives in
the SealedSecret, as the `PAPERLESS_SOCIALACCOUNT_PROVIDERS` JSON:

```json
{"openid_connect": {"SCOPE": ["openid", "profile", "email"],
  "OAUTH_PKCE_ENABLED": true,
  "APPS": [{"provider_id": "authelia", "name": "Authelia",
            "client_id": "paperless", "secret": "<plaintext>",
            "settings": {"server_url": "https://auth.yourdomain.com",
                         "token_auth_method": "client_secret_basic"}}]}}
```

`provider_id: authelia` is what forms the `oidc/authelia/` segment of the callback —
change one, change both. `PAPERLESS_APPS=allauth.socialaccount.providers.openid_connect`
(plain env in the Deployment) is what activates the provider at all.

!!! warning "Gate the seal on reading the Secret back"
    The `secret` field is buried *inside* the JSON blob, and it's the substitution
    everyone misses when filling in the seal command (ask how we know). The failure
    is maddeningly indirect: Authelia validates the client and redirect fine, then
    every token exchange dies `invalid_client` no matter what digest is registered.
    Before testing login, read the live value back and eyeball the `secret` field:

    ```bash
    kubectl get secret -n paperless paperless-secrets \
      -o jsonpath='{.data.PAPERLESS_SOCIALACCOUNT_PROVIDERS}' | base64 -d
    ```

    Two restart rules while iterating here: Authelia reads its config **at startup
    only** (a values/ConfigMap sync does *not* roll the StatefulSet — `kubectl
    rollout restart statefulset -n authelia authelia`), and the same goes for
    paperless env after a Secret change (`kubectl rollout restart deployment -n
    paperless paperless`).

!!! warning "Link the admin account after first OIDC login"
    An OIDC login that doesn't match an existing paperless user **creates a new
    user**. Log in once as the local `admin`, then **My Profile → Connected social
    accounts → Connect** to bind your Authelia identity to it. Leave
    `PAPERLESS_DISABLE_REGULAR_LOGIN` unset until OIDC has round-tripped; flip it
    (and `PAPERLESS_REDIRECT_LOGIN_TO_SSO`) later if you want SSO-only.

## Step 4 — Routing

`paperless` is already present in the Cloudflare module's `var.services` (pre-staged),
so DNS needs **no change** — just confirm the record resolves. The `HTTPRoute` attaches
to the shared Gateway's `websecure` listener as usual; no middleware
([pattern, Step 4](apps-deploy-pattern.md)).

## Step 5 — Register the Application and sync

`bootstrap/paperless.yaml`: single Application, `prune: false` (document archive — a
path typo must not tear down PVCs), client-side apply (HTTPRoute + SSA = permanent
OutOfSync), `CreateNamespace`. Branch → PR → merge; ArgoCD does the rest. Note the
Authelia values change rolls Authelia itself — do that merge **before** testing login.

## Scan-to-archive

Anything that lands in the consume volume is OCR'd, indexed, and filed within ~a
minute (60 s poll + processing). Quickest test path:

```bash
kubectl cp ./test.pdf paperless/$(kubectl get pod -n paperless -l app=paperless -o jsonpath='{.items[0].metadata.name}'):/usr/src/paperless/consume/
```

Pointing a network scanner or phone scan app at the consume folder needs a share
(SMB on the NAS, synced in, or an NFS mount of the consume PV's backing dir on
topaz). That wiring is deliberately out of scope here — decide it when a real
scanner exists.

## Verification

- [ ] `kubectl get pods -n paperless` — `paperless` and `paperless-redis` Running
- [ ] ArgoCD: `paperless` Synced / Healthy
- [ ] `https://paperless.yourdomain.com` loads over the wildcard cert; local `admin` login works
- [ ] OIDC round-trips: logout → *Log in with Authelia* → 2FA → back in as the linked user
- [ ] Drop a test PDF in the consume dir → document appears with OCR'd, searchable text
- [ ] **velero gate**: after the next 04:00 UTC run, the `PodVolumeBackup`s for
      `paperless-data` / `paperless-media` show **non-zero bytes** — never trust
      `Completed`:

    ```bash
    kubectl -n velero get podvolumebackups \
      -o custom-columns=NAME:.metadata.name,POD:.spec.pod.name,BYTES:.status.progress.bytesDone \
      | grep paperless
    ```

- [ ] **DB backup gate**: after the next 04:30 NAS run, `paperless-<date>.dump`
      exists in the Garage `postgres-backups` bucket with a non-trivial size
      (list with rclone via the `garage-pg` remote, per Runbook 27)
