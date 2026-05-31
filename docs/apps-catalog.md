# App Catalog — Simple Services

These services follow the [app deployment pattern](apps-deploy-pattern.md) exactly — workload (Helm or raw), `SealedSecret`, an `HTTPRoute` on the shared Gateway, and Authelia. Instead of a full runbook each, this page records only the **per-app deltas** that feed the pattern, plus the one or two non-obvious gotchas. The deployed truth for each is `homelab-manifests/apps/<app>/`.

Services with real deployment complexity keep their own runbook (Arr stack, Authelia, Immich, Home Assistant, Ollama, ntfy).

## linkding

Bookmark manager with a browser extension and OIDC login.

| Field | Value |
|---|---|
| Workload | raw manifests — image `sissbruecker/linkding` (pin a tag) |
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
| Workload | raw manifests — image `ghcr.io/mealie-recipes/mealie` (pin a tag) |
| Namespace / hostname | `mealie` / `recipes.yourdomain.com` |
| Service port | 9000 |
| Storage | `nfs-storage`, 5 Gi |
| Secret keys | `OIDC_CLIENT_SECRET` |
| Auth | OIDC client (own user system, group-based) |

**Gotchas**

- Mealie v2 needs a **confidential client** (with a secret) on the Authorization Code flow.
- **Two** redirect URIs are required: `…/login` and `…/login?direct=1` (the second is for `OIDC_AUTO_REDIRECT=true`).
- The default admin (`changeme@example.com` / `MyPassword`) stays active until you log in via OIDC and promote your user to admin (Settings → Users), then set `ALLOW_PASSWORD_LOGIN=false`.

## Uptime Kuma

Uptime monitoring with status pages and ntfy alerts.

| Field | Value |
|---|---|
| Workload | raw manifests — image `louislam/uptime-kuma:2` |
| Namespace / hostname | `uptime-kuma` / `status.yourdomain.com` |
| Service port | 3001 |
| Storage | **`local-path`, 1 Gi** (see gotcha) |
| Secret keys | none (ntfy credentials entered in the UI) |
| Auth | Authelia **ForwardAuth** (no built-in SSO) |

**Gotchas**

- **Do not use `nfs-storage`.** Uptime Kuma uses SQLite, which needs POSIX file locking; NFS doesn't provide it reliably and will corrupt the DB. Use `local-path` with `strategy: Recreate` (the volume mounts on a single node).
- ForwardAuth puts an Authelia login in front of Uptime Kuma's own login — see the ForwardAuth section of the [deployment pattern](apps-deploy-pattern.md) (note: not yet verified against a live cluster).
- Notifications: Settings → Notifications → ntfy, server `https://ntfy.yourdomain.com`, topic `homelab-alerts`.
