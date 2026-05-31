# App Catalog — Simple Services

These services follow the [app deployment pattern](apps-deploy-pattern.md) exactly — workload (Helm or raw), `SealedSecret`, an `HTTPRoute` on the shared Gateway, and Authelia. Instead of a full runbook each, this page records only the **per-app deltas** that feed the pattern, plus the one or two non-obvious gotchas. The deployed truth for each is `homelab-manifests/apps/<app>/`.

Services with real deployment complexity keep their own runbook: a database (Nextcloud, Paperless, BookStack), multiple components (Arr stack, Reactive Resume), a non-cluster model (Immich, Home Assistant, Syncthing, Ollama), a non-HTTP service (RustDesk), config-heavy reference (Homepage), or auth backbones (Authelia, ntfy).

!!! note "All images below are shown as the upstream repo — pin a specific tag"
    The source runbooks used `:latest`; pin an explicit version in the manifest and let Renovate bump it.

## Actual Budget

Personal budgeting (envelope method) with OIDC login.

| Field | Value |
|---|---|
| Workload | raw manifests — `actualbudget/actual-server` |
| Namespace / hostname | `actual-budget` / `budget.yourdomain.com` |
| Service port | 5006 |
| Storage | `nfs-storage`, 1 Gi |
| Secret keys | `ACTUAL_OPENID_CLIENT_SECRET` |
| Auth | OIDC client |

**Gotchas**

- OIDC support in Actual is **in preview** — enabled via `ACTUAL_OPENID_*` env, and the config surface can change between releases. Verify against current Actual docs at deploy.

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
| Workload | raw manifests — `collabora/code` |
| Namespace / hostname | `collabora` / `office.yourdomain.com` |
| Service port | 9980 |
| Storage | none (stateless) |
| Secret keys | none |
| Auth | **None** — Nextcloud reaches it server-side via WOPI |

**Gotchas**

- **Do not put Authelia/ForwardAuth in front of it.** Nextcloud's server calls Collabora directly (WOPI); an auth challenge breaks document editing. (This is the one app where the route stays unauthenticated.)
- Pairs with Nextcloud (R13): enable the Nextcloud **Office** app and point it at `https://office.yourdomain.com`. Set Collabora's allowed-host/`aliasgroup` to the Nextcloud hostname.

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

## FreshRSS

Self-hosted RSS/Atom feed aggregator with OIDC login.

| Field | Value |
|---|---|
| Workload | raw manifests — `freshrss/freshrss` |
| Namespace / hostname | `freshrss` / `rss.yourdomain.com` |
| Service port | 80 |
| Storage | `nfs-storage`, 2 Gi |
| Secret keys | `OIDC_CLIENT_SECRET`, `OIDC_CLIENT_CRYPTO_KEY` |
| Auth | OIDC client |

**Gotchas**

- OIDC callback **requires a trailing slash**.
- Pick the correct authentication method **in the setup wizard** — choosing the wrong mode there can lock you out and is awkward to undo.

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
| Workload | raw manifests — `sissbruecker/linkding` |
| Namespace / hostname | `linkding` / `bookmarks.yourdomain.com` |
| Service port | 9090 |
| Storage | `nfs-storage`, 1 Gi |
| Secret keys | `OIDC_RP_CLIENT_SECRET`, `LD_SUPERUSER_PASSWORD` |
| Auth | OIDC client (own user system — **not** ForwardAuth) |

**Gotchas**

- OIDC callback **requires a trailing slash**: `https://bookmarks.yourdomain.com/oidc/callback/`.
- Enable OIDC via env (`LD_ENABLE_OIDC=True`, the `OIDC_OP_*` endpoint URLs, `OIDC_USE_PKCE=True`). The `admin` superuser remains as a local-login fallback.
- For the browser extension, generate an API token in the UI (Profile → API token) after first login.

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
| Storage | **`local-path`, 1 Gi** (see gotcha) |
| Secret keys | none (ntfy credentials entered in the UI) |
| Auth | Authelia ForwardAuth (no built-in SSO) |

**Gotchas**

- **Do not use `nfs-storage`.** Uptime Kuma uses SQLite, which needs POSIX file locking; NFS doesn't provide it reliably and will corrupt the DB. Use `local-path` with `strategy: Recreate` (the volume mounts on a single node).
- Notifications: Settings → Notifications → ntfy, server `https://ntfy.yourdomain.com`, topic `homelab-alerts`.

## Vikunja

To-do / task management with lists, labels, and OIDC login.

| Field | Value |
|---|---|
| Workload | raw manifests — `vikunja/vikunja` |
| Namespace / hostname | `vikunja` / `tasks.yourdomain.com` |
| Service port | 3456 |
| Storage | `nfs-storage` (files ~1 Gi + data ~2 Gi) |
| Secret keys | `VIKUNJA_AUTH_OPENID_PROVIDERS_AUTHELIA_CLIENTSECRET` |
| Auth | OIDC client |

**Gotchas**

- Redirect URI must match Vikunja's format (`…/auth/openid/authelia`, where `authelia` is the provider key).
- Configure the provider with the Authelia OIDC **discovery** URL plus that provider key.
