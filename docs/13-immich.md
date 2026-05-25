# Runbook 13: Immich

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

The [R10 backup script](10-backups.md#backup-script) already includes this for routine snapshots.

## Traefik IngressRoute (optional, for HTTPS on the cluster domain)

If you want `https://immich.yourdomain.com` to resolve through Traefik on the cluster (instead of `http://<nas-ip>:2283`), expose Immich's web port via a Kubernetes ExternalName service pointing at the NAS, then add an IngressRoute. Otherwise just access it via NAS IP.

## Verification

- [ ] On the NAS:

    ```bash
    docker compose ps    # all immich_* containers healthy
    ```

- [ ] `https://immich.yourdomain.com` (or `http://<nas-ip>:2283`) loads the welcome screen.
- [ ] Upload a test photo via the web UI — it appears in the timeline within a few seconds.
- [ ] Machine-learning job ran (Search → Search by keyword in photo content) — returns results after a few minutes of background processing.
