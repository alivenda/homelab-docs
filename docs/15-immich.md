# Runbook 15: Immich

Self-hosted photo and video management.

| | |
|---|---|
| **Difficulty** | Beginner–Intermediate |
| **Time Estimate** | 1 hour |
| **Runs On** | NAS (Docker) — strongly recommended |
| **Min Requirements** | 6 GB RAM, 2 CPU cores |

!!! warning "Not on the cluster"
    Immich officially requires 6 GB RAM minimum and is CPU-intensive. This runbook intentionally keeps Immich on the NAS via Docker — not on the k3s cluster. CM4 nodes lack the RAM, the photo library belongs near your bulk storage, and machine-learning workloads benefit from the NAS's larger CPU. Do not migrate this to Helm/k3s without a hardware upgrade.

## Install

```bash
mkdir ~/immich && cd ~/immich
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```

**Before** you run `docker compose up`, edit `.env` and set at minimum:

| Variable | What to set | Why |
|---|---|---|
| `UPLOAD_LOCATION` | Absolute path on the NAS, e.g. `/volume1/immich/library` | Where photos are stored. Must exist before launch. |
| `DB_PASSWORD` | Strong random password | Postgres password. Generate with `openssl rand -base64 24`. Save to Vaultwarden. |
| `IMMICH_VERSION` | Pin a release tag (e.g. `v1.107.0`) | Avoid `latest` so upgrades are deliberate. |
| `TZ` | Your timezone, e.g. `America/New_York` | Affects scheduled jobs and timestamps. |

```bash
docker compose up -d
```

## Backup before every update

```bash
docker exec -t immich_postgres pg_dumpall -c -U postgres \
  > immich_backup_$(date +%Y%m%d).sql
```

Immich's database is deliberately **not** covered by the shared NAS Postgres backup job —
its bundled Postgres predates the [shared server](27-nas-postgres.md) and stays separate. The
command above is the manual capture to run **before an update**. Routine off-box protection is
automated separately: Immich's own built-in backup dumps the database nightly to
`${UPLOAD_LOCATION}/backups/`, and the [Immich database → Garage](10-backups.md#immich-database-garage)
sync job ships those dumps to a dedicated Garage bucket so they survive a NAS volume failure
and land in the [cold-shutdown export](cold-shutdown.md).

## Traefik HTTPRoute (optional, for HTTPS on the cluster domain)

To resolve `https://immich.yourdomain.com` through Traefik on the cluster (instead of `http://<nas-ip>:2283`), front the NAS-hosted Immich with a selector-less `Service` + a manual `EndpointSlice` pointing at the NAS, then attach an `HTTPRoute`. This is exactly how `homelab-manifests/apps/immich/manifests/` already exposes it — TLS is handled by the Gateway's wildcard cert (see [Deploying an App](apps-deploy-pattern.md)):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: immich-nas
  namespace: immich
spec:
  ports:
    - port: 2283
      targetPort: 2283
---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: immich-nas
  namespace: immich
  labels:
    kubernetes.io/service-name: immich-nas
addressType: IPv4
endpoints:
  - addresses: ["<nas-ip>"]
    conditions:
      ready: true
ports:
  - port: 2283
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: immich
  namespace: immich
spec:
  parentRefs:
    - name: traefik
      namespace: traefik
      sectionName: websecure
  hostnames:
    - immich.yourdomain.com
  rules:
    - backendRefs:
        - name: immich-nas
          port: 2283
```

Otherwise just access it via the NAS IP.

## Verification

- [ ] On the NAS:

    ```bash
    docker compose ps    # all immich_* containers healthy
    ```

- [ ] `https://immich.yourdomain.com` (or `http://<nas-ip>:2283`) loads the welcome screen.
- [ ] Upload a test photo via the web UI — it appears in the timeline within a few seconds.
- [ ] Machine-learning job ran (Search → Search by keyword in photo content) — returns results after a few minutes of background processing.
