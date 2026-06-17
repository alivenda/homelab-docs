# App Catalog — Simple Services

These services follow the [app deployment pattern](apps-deploy-pattern.md) exactly — workload (Helm or raw), `SealedSecret`, an `HTTPRoute` on the shared Gateway, and Authelia. Instead of a full runbook each, this page records only the **per-app deltas** that feed the pattern, plus the one or two non-obvious gotchas. The deployed truth for each is `homelab-manifests/apps/<app>/`.

Services with real deployment complexity keep their own runbook: a database (Nextcloud, Paperless, BookStack, Vikunja, Miniflux), multiple components (Arr stack, Reactive Resume), a non-cluster model (Immich, Home Assistant, Syncthing, Ollama), a non-HTTP service (RustDesk), config-heavy reference (Homepage), or auth backbones (Authelia, ntfy).

!!! note "All images below are shown as the upstream repo — pin a specific tag"
    The source runbooks used `:latest`; pin an explicit version in the manifest and let Renovate bump it.

## Actual Budget

Personal budgeting (envelope method) with OIDC login.

| Field | Value |
|---|---|
| Workload | raw manifests — `actualbudget/actual-server` (GitHub repo archived 2025-02-10; server now in the `actualbudget/actual` monorepo, image still published to `actualbudget/actual-server`) |
| Namespace / hostname | `actual-budget` / `budget.yourdomain.com` |
| Service port | 5006 |
| Storage | `local-path`, 2 Gi — SQLite (`account.sqlite` + per-budget `db.sqlite` + `server-files`/`user-files`) under `/data`; **not** NFS (corrupts SQLite) |
| Secret keys | `ACTUAL_OPENID_CLIENT_SECRET` |
| Auth | OIDC client (own user system — **not** ForwardAuth) |

**Gotchas**

- OIDC support in Actual is **in preview** — enabled via `ACTUAL_OPENID_*` env, and the config surface can change between releases. Verify against current Actual docs **and the server source** at deploy. Everything below was verified against actual-server **v26.6.0** (`packages/sync-server/src/accounts/openid.ts` + `load-config.js`).
- Config is **env-only** — no ConfigMap. The server writes its own `/data/config.json`; there is no settings-override file (the linkding contrast).
- Storage is SQLite, so **local-path, never `nfs-storage`** — the catalog originally listed NFS, which corrupts SQLite. One PVC at `/data` covers the whole data tree.
- OIDC **supports discovery**: set `ACTUAL_OPENID_DISCOVERY_URL` to the Authelia issuer base (`https://auth.yourdomain.com`); `Issuer.discover()` appends `/.well-known/openid-configuration` (no need to spell out the four endpoints, the linkding contrast).
- The redirect is **built from `ACTUAL_OPENID_SERVER_HOSTNAME`**, not the request scheme — `https://budget.yourdomain.com/openid/callback` (**no** trailing slash) — so there is no TLS-proxy `http://` redirect bug, and the Authelia `redirect_uris` must match exactly.
- The authorization request **sends PKCE S256** (`require_pkce: true`), and the token exchange uses `openid-client`'s default **`client_secret_basic`** — not `client_secret_post`.
- Username = `preferred_username` from the **userinfo** endpoint, so the `profile` scope is required. No group→role mapping → no `groups` scope, no `claims_policy`.
- The **first user to sign in via OIDC becomes the server owner**. `ACTUAL_USER_CREATION_MODE=manual` (default) means the owner must add every other user in-app (matching username) before they can log in; `login` would auto-provision any Authelia user.
- **Break-glass — OpenID-only**: the first owner bootstraps through Authelia, so there's no password login, and Actual's preview auth can't run password + OpenID at once — you **can't** add a parallel password (upstream [#5248](https://github.com/actualbudget/actual/issues/5248)). If Authelia is down, flip in-pod with `node /app/src/scripts/reset-password.js` (→ password-only), then `node /app/src/scripts/enable-openid.js` to flip back. `ACTUAL_OPENID_ENFORCE=false` (default) is kept so the flip needs no env change, and it dodges upstream [#6333](https://github.com/actualbudget/actual/issues/6333) (enforce silently ignored when the provider is unreachable at boot).
- Health probe = `/health` (returns `{status:'UP'}` 200, unauthenticated, no Host header). **No metrics endpoint** — no ServiceMonitor.

## Audiobookshelf

Audiobook + podcast server with native iOS/Android apps and a browser player.
Verified against **v2.35.1** (latest stable GitHub release; the GHCR tag is a
multi-arch index with `linux/arm64` confirmed — the old "amd64-only" claim is
stale).

| Field | Value |
|---|---|
| Workload | raw manifests — `ghcr.io/advplyr/audiobookshelf:2.35.1` |
| Namespace / hostname | `audiobookshelf` / `audiobooks.yourdomain.com` |
| Service port | 13378 (image default `PORT=80` is privileged → moved so it runs non-root) |
| Storage | `/config` (SQLite) + `/metadata` on `local-path` app-state node; library `/audiobooks` + `/podcasts` on a **static NFS PV → NAS** `10.0.20.50` |
| Secret keys | none (OIDC client secret entered in the ABS UI — the Immich model) |
| Auth | ABS-native **OIDC** (Authelia), route unauthenticated — **not** ForwardAuth |
| Health | `GET /healthcheck` (200) + `GET /ping` (`{"success":true}`), both unauthenticated |

**Gotchas**

- **Not ForwardAuth** (the old row was wrong). The mobile apps + browser player
  hit ABS's API with bearer tokens; a Authelia browser challenge in front of the
  API breaks them — same exclusion bucket as Vaultwarden / Home Assistant / ntfy.
  Use ABS-native OIDC instead and leave the route open. From the Authelia ABS
  integration guide: PKCE **S256 required** (enable the PKCE toggle in ABS too),
  `client_secret_basic`, `groups` scope (→ `claims_policy: default`), and **three**
  redirect URIs — `…/auth/openid/callback`, `…/auth/openid/mobile-redirect`, and
  the mobile custom scheme **`audiobookshelf://oauth`** (the one that's easy to
  miss and is what makes the mobile-app login work). ABS uses OIDC discovery.
- **Three storage tiers, and the SQLite trap.** `/config` holds the live SQLite
  DB and per the ABS docs "needs to be on the same machine" → `local-path`, **not**
  `nfs-storage` (NFS corrupts SQLite). `/metadata` (covers/cache/backups, **no
  live DB**) sits on `local-path` alongside it. The **library** is bulk media on
  the NAS array over NFS — a *static* PV bound by claimRef, `storageClassName ""`,
  not the dynamic `nfs-storage` SSD (same split as the arr `/data` export).
- **Runs as root by default with no PUID/PGID** (that's a LinuxServer convention
  this image lacks). Run it as `securityContext` uid/gid 1000 — required so it can
  write the `1000:1000`-owned NAS export under NFS `root_squash` — and set
  `PORT=13378` so a non-root process can bind. `fsGroupChangePolicy: OnRootMismatch`
  + a pre-owned export root keeps fsGroup from recursively chowning the library.
- **NAS export is a pre-deploy prerequisite.** Create the export (`audiobooks/` +
  `podcasts/` subdirs, owned `1000:1000`, RW to node IPs `10.0.20.10-13`) and set
  the PV's `nfs.path` to match before first sync.
- **Websockets**: progress sync is Socket.IO on `/socket.io/`; Traefik upgrades it
  transparently over the HTTPRoute — nothing to configure.
- **Backup gate** (normal): `PodVolumeBackup` bytes > 0 for the `config` **and**
  `metadata` pod-volumes; the NAS library is backed up NAS-side, not by velero.
- **Break-glass** = the local root account created during ABS setup; configure
  OIDC after it and leave Auto-register on so SSO users provision on first login.

## Collabora Online

Online office suite (CODE) — the editing backend for Nextcloud.

| Field | Value |
|---|---|
| Workload | raw manifests — `collabora/code` (official image is multi-arch; arm64 runs on the CM4s) |
| Namespace / hostname | `collabora` / `office.yourdomain.com` |
| Service port | 9980 |
| Storage | none (stateless) |
| Secret keys | none |
| Auth | **None** — Nextcloud reaches it server-side via WOPI |

**Gotchas**

- **Do not put Authelia/ForwardAuth in front of it.** Nextcloud's server calls Collabora directly (WOPI), and the browser opens a websocket straight to coolwsd; an auth challenge on either leg breaks document editing. (This is the one app where the route stays unauthenticated.)
- The compensating control for the open route is the WOPI host pin: `aliasgroup1=https://nextcloud.yourdomain.com:443` — coolwsd rejects WOPI traffic for any other origin. Plus: leave the admin console's `username`/`password` env **unset**, which disables the console entirely (official-docs behaviour) — nothing to brute-force on the open route.
- The image's entrypoint runs coolwsd with `--use-env-vars`, so config is plain env. Behind the TLS-terminating Gateway the trio is `extra_params=--o:ssl.enable=false --o:ssl.termination=true`, `server_name=office.yourdomain.com` (responses must carry the external hostname or WOPI handshakes fail), and `DONT_GEN_SSL_CERT=1`.
- Pairs with Nextcloud (R13): enable the Nextcloud **Office** app (richdocuments) and point it at `https://office.yourdomain.com`. Smoke test before touching Nextcloud: `https://office.yourdomain.com/hosting/discovery` must return the WOPI capability XML.
- Stateless means the backup check **inverts**: the velero gate is the *absence* of any `PodVolumeBackup` for the namespace — if one appears, something grew state that shouldn't exist.

## Donetick

Recurring chores & habits tracker with OIDC login (single Go binary serving the
React PWA + API). Verified against **v0.1.75** (latest stable; newer tags are
`v0.1.76-beta.*`), `linux/arm64` confirmed.

| Field | Value |
|---|---|
| Workload | raw manifests — `donetick/donetick:v0.1.75` |
| Namespace / hostname | `donetick` / `chores.yourdomain.com` |
| Service port | 2021 |
| Storage | `local-path`, 2 Gi, app-state node — SQLite (**not** `nfs-storage`) |
| Config | ConfigMap `selfhosted.yaml` subPath-mounted at `/config/selfhosted.yaml` |
| Secret keys | `DT_JWT_SECRET`, `DT_OAUTH2_CLIENT_SECRET` (env, override the file) |
| Auth | OIDC client (Authelia), no ForwardAuth |
| Health | `GET /api/v1/health` (200, unauthenticated) |

**Gotchas**

- **SQLite → `local-path`, not `nfs-storage`** (the old row was wrong): NFS lacks
  reliable POSIX locking and corrupts SQLite. One file `donetick.db`; the path is
  set by `DT_SQLITE_PATH` env (read via `os.Getenv`, **not** a YAML key) pointed
  at the PVC. Normal velero bytes-gate.
- **Config is a mounted file, not pure env.** Most keys are `DT_*`-overridable
  (`viper.AutomaticEnv` + `ExperimentalBindStruct`), but `oauth2.scopes` is a
  YAML list env can't express — so non-secret config lives in a ConfigMap and
  only the two secrets come from env.
- **OAuth2 endpoints are explicit** (authorization / token / userinfo URLs), not
  a discovery URL. Redirect is the *frontend* page `…/auth/oauth2` (no trailing
  slash). **No PKCE**; token auth = `client_secret_basic`. Username + email both
  come from the `email` claim, display name from `name` (so `profile` scope is
  required).
- **`enableServiceLinks: false`** — the `donetick` Service injects `DONETICK_*`
  env that overlaps the `DONETICK_*` vars the app reads.
- **Break-glass = local admin.** `is_user_creation_disabled` closes only the
  local *sign-up* form; the local *login* stays as the Authelia-down fallback,
  and OIDC keeps auto-provisioning (the callback never checks the flag). Deploy
  with it `false`, register one local admin + log in via OIDC, then flip to
  `true`.
- **Realtime SSE + websocket** on the same origin (`/api/v1/...`) pass through
  the one HTTPRoute (Traefik upgrades the socket transparently); don't put
  ForwardAuth in front — it breaks both login and realtime.
- Optional OIDC group→role mapping (`admin_groups`/`manager_groups`) reads groups
  from the **ID token**, so it would need the `groups` scope + an Authelia
  `claims_policy` — left unused here.

## Kavita

Manga / comics / e-book reader.

| Field | Value |
|---|---|
| Workload | raw manifests — `jvmilazz0/kavita` |
| Namespace / hostname | `kavita` / `books.yourdomain.com` |
| Service port | 5000 |
| Storage | config on `nfs-storage` (~2 Gi); large media PVC (~500 Gi) for the library |
| Secret keys | none (OIDC configured in the UI) |
| Auth | OIDC — configured in Kavita's **web UI**, not env |

**Gotchas**

- OIDC is set up through Settings in the UI; a **restart is required** after enabling it.

## linkding

Bookmark manager with a browser extension and OIDC login.

| Field | Value |
|---|---|
| Workload | raw manifests — `sissbruecker/linkding` (standard variant) |
| Namespace / hostname | `linkding` / `bookmarks.yourdomain.com` |
| Service port | 9090 |
| Storage | `local-path`, 2 Gi — SQLite + favicons/previews under `/etc/linkding/data`; **not** NFS (corrupts SQLite) |
| Secret keys | `OIDC_RP_CLIENT_SECRET`, `LD_SUPERUSER_PASSWORD` |
| Auth | OIDC client (own user system — **not** ForwardAuth) |

**Gotchas**

- Use the **standard** image variant, not `-plus` — `-plus` bundles Chromium for HTML snapshot archiving, far too heavy for Pi-class nodes.
- linkding cannot detect a TLS-terminating proxy, so the OIDC `redirect_uri` goes out as `http://` and the provider rejects it ([linkding#1366](https://github.com/sissbruecker/linkding/issues/1366)). No env option exists; mount a one-line settings override (the image ships `bookmarks/settings/custom.py` as a placeholder that `prod.py` star-imports last) via ConfigMap `subPath`: `SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")`.
- OIDC is mozilla-django-oidc: **no discovery** — set all four `OIDC_OP_*` endpoint URLs explicitly. The callback **requires a trailing slash** (`https://bookmarks.yourdomain.com/oidc/callback/`), PKCE is on by default (S256), and the token request carries the secret in the POST body — the Authelia client needs `token_endpoint_auth_method: 'client_secret_post'`, not `client_secret_basic`.
- Set `OIDC_USERNAME_CLAIM=preferred_username` — the default (`email`) turns usernames into full email addresses. Accounts auto-create on first login as regular users; the `admin` superuser remains as a local-login fallback (flip `LD_DISABLE_LOGIN_FORM=True` once the round-trip is verified).
- Setting `HOST_NAME` makes Django's `ALLOWED_HOSTS` strict — kubelet probes to `/health` must then send an explicit `Host:` header.
- No metrics endpoint — no ServiceMonitor.
- For the browser extension, generate the API token in the UI (Settings → Integrations) after first login.

## Mealie

Recipe manager with meal planning and OIDC login.

| Field | Value |
|---|---|
| Workload | raw manifests — `ghcr.io/mealie-recipes/mealie` |
| Namespace / hostname | `mealie` / `recipes.yourdomain.com` |
| Service port | 9000 |
| Storage | `nfs-storage`, 5 Gi |
| Secret keys | `OIDC_CLIENT_SECRET` |
| Auth | OIDC client (own user system, group-based) |

**Gotchas**

- Mealie v2 needs a **confidential client** (with a secret) on the Authorization Code flow.
- **Two** redirect URIs are required: `…/login` and `…/login?direct=1` (the second is for `OIDC_AUTO_REDIRECT=true`).
- The default admin (`changeme@example.com` / `MyPassword`) stays active until you log in via OIDC and promote your user to admin (Settings → Users), then set `ALLOW_PASSWORD_LOGIN=false`.

## TriliumNext Notes

Hierarchical note-taking with a server + desktop/web sync.

| Field | Value |
|---|---|
| Workload | raw manifests — `triliumnext/notes` |
| Namespace / hostname | `trilium` / `notes.yourdomain.com` |
| Service port | 8080 |
| Storage | `nfs-storage`, 5 Gi |
| Secret keys | none |
| Auth | Authelia ForwardAuth |

**Gotchas**

- Pin the image tag — TriliumNext moves fast and the sync protocol version must match between the server and the desktop clients.

## Uptime Kuma

Uptime monitoring with status pages and ntfy alerts.

| Field | Value |
|---|---|
| Workload | raw manifests — `louislam/uptime-kuma:2` |
| Namespace / hostname | `uptime-kuma` / `status.yourdomain.com` |
| Service port | 3001 |
| Storage | **`local-path`, 2 Gi** (see gotcha) |
| Secret keys | none (ntfy credentials entered in the UI) |
| Auth | Authelia ForwardAuth (no built-in SSO) |

**Gotchas**

- **Do not use `nfs-storage`.** Uptime Kuma uses SQLite, which needs POSIX file locking; NFS doesn't provide it reliably and will corrupt the DB. Use `local-path` with `strategy: Recreate` (the volume mounts on a single node).
- Notifications: Settings → Notifications → ntfy, server `https://ntfy.yourdomain.com`, topic `uptime`, auth = *Bearer token* (the `uptime-kuma` publisher token — see Runbook 19; the account is write-only on its own topic).
