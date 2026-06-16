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

Audiobook and podcast server with apps for iOS/Android.

| Field | Value |
|---|---|
| Workload | raw manifests — `ghcr.io/advplyr/audiobookshelf` |
| Namespace / hostname | `audiobookshelf` / `audiobooks.yourdomain.com` |
| Service port | 80 |
| Storage | config/metadata on `nfs-storage` (~5 Gi); **separate large media PVCs** (100–500 Gi) for audiobooks/podcasts |
| Secret keys | none |
| Auth | Authelia ForwardAuth (no built-in SSO) |

**Gotchas**

- The mobile apps can't complete the Authelia browser login, so ForwardAuth blocks them. Either exempt the API path from ForwardAuth or front only the web UI — confirm the approach when the first ForwardAuth app is deployed.
- Keep media on its own large PVC, separate from the small config volume.

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

Shared chore and task tracker with OIDC login.

| Field | Value |
|---|---|
| Workload | raw manifests — `donetick/donetick` |
| Namespace / hostname | `donetick` / `chores.yourdomain.com` |
| Service port | 2021 |
| Storage | `nfs-storage`, 1 Gi |
| Secret keys | `JWT_SECRET`, OAuth client secret |
| Auth | OIDC client |

**Gotchas**

- Donetick needs the OAuth2 endpoints set **explicitly** (authorization / token / userinfo URLs), not just a discovery URL.

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
