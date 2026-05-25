# Runbook 10: Backups and Disaster Recovery

Local backups with a clearly-marked offsite TODO.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 2–4 hours |
| **Runs On** | NAS (primary backup target), cluster nodes |

## Strategy

| Tier | What | Where |
|---|---|---|
| Tier 1 — Critical | Vaultwarden, Immich DB, Nextcloud DB, Paperless DB, k3s manifests | NAS daily + retained 90d |
| Tier 2 — Important | Service configs, ArgoCD state, Grafana dashboards | NAS daily + retained 30d |
| Tier 3 — Replaceable | Container images, media | NAS weekly + retained 14d |

The strategy above covers application data (DB dumps) and IaC (manifests). It does NOT cover Kubernetes objects themselves — PVCs, CRDs, secrets in non-Git-tracked namespaces, helm release state. For that, install Velero.

## Velero for k8s-native PVC backup

Velero with the restic backend snapshots PVC contents and stores them in an S3-compatible target. The simplest target is a MinIO container on your NAS.

### Stand up MinIO on the NAS

```yaml
# /volume1/docker/minio/docker-compose.yml
services:
  minio:
    image: minio/minio:latest
    container_name: minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: <USER>
      MINIO_ROOT_PASSWORD: <STRONG_PASSWORD>
    volumes:
      - /volume1/minio-data:/data
    ports:
      - "9000:9000"   # S3 API
      - "9001:9001"   # web console
    restart: unless-stopped
```

```bash
docker compose -f /volume1/docker/minio/docker-compose.yml up -d
```

Visit `http://<nas-ip>:9001`, log in, create a bucket named `homelab`.

### Create the Velero credentials file

```bash
# On your machine (wherever you'll run the helm commands)
cat > minio-creds <<EOF
[default]
aws_access_key_id = <USER>
aws_secret_access_key = <STRONG_PASSWORD>
EOF
```

### Install Velero

The Velero chart's schema changed in v3.0.0+: `backupStorageLocation` is now an array, `provider` lives inside each entry, and the old `deployRestic` flag was renamed `deployNodeAgent`.

`velero-values.yaml`:

```yaml
configuration:
  backupStorageLocation:
    - name: default
      provider: aws       # use 'aws' provider for any S3-compatible target
      bucket: homelab
      default: true
      config:
        region: minio
        s3Url: http://10.0.20.50:9000  # NAS (R2 static-IP table)
        s3ForcePathStyle: "true"

deployNodeAgent: true       # replaces deployRestic; needed for PVC-content backups

credentials:
  useSecret: true
  # secretContents.cloud comes in via --set-file at install time
```

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts

helm upgrade --install velero vmware-tanzu/velero \
  --version <X.Y.Z> \
  --namespace velero --create-namespace \
  --values velero-values.yaml \
  --set-file credentials.secretContents.cloud=./minio-creds

# Verify
velero backup-location get

# Take a full cluster backup
velero backup create homelab-$(date +%Y%m%d)
```

Pin `--version` to a current release listed on [vmware-tanzu/helm-charts](https://github.com/vmware-tanzu/helm-charts/tree/main/charts/velero).

!!! tip "Off-site backup target"
    If you don't want to run MinIO on the NAS, Velero supports any S3-compatible target (Backblaze B2, Wasabi, Cloudflare R2) for off-site storage. R2's free tier is generous for homelab volumes — point the same Velero config at R2 and you get cloud-hosted backups for free.

!!! warning "Off-site backup is still a TODO"
    Local backups protect against disk failure but NOT fire/theft/flood. Options: friend's house with Tailscale + Restic, cycled external drives, second location you control. Schedule a calendar reminder within 3 months.

## Restic Setup

Restic backs up filesystem-level data: DB dumps, config files, k3s server manifests. Run this **on a host with access to the cluster databases** — typically ruby (where `kubectl` works) and the NAS (where mount happens).

```bash
# On ruby
apt install -y restic
mkdir -p /mnt/backups
mount -t nfs <NAS_IP>:/volume1/backups /mnt/backups
export RESTIC_REPOSITORY=/mnt/backups/restic
export RESTIC_PASSWORD=<STRONG_REPO_PASSWORD>
restic init
```

!!! warning
    Store the `RESTIC_PASSWORD` in Vaultwarden AND on paper in a fireproof safe. If you lose it, all backups are permanently unrecoverable.

## Backup Script

Run on ruby (where `kubectl` works against the cluster). Comment out the lines for services you haven't deployed yet — pulling a DB from a pod that doesn't exist will fail the whole script under `set -euo pipefail`.

```bash
#!/bin/bash
set -euo pipefail
export RESTIC_REPOSITORY=/mnt/backups/restic
export RESTIC_PASSWORD_FILE=/root/.restic-password
BACKUP_TMP=/tmp/homelab-backup
mkdir -p "$BACKUP_TMP"

# --- Per-service DB dumps (comment out any you haven't deployed) ---

# Vaultwarden (SQLite)
if kubectl -n vaultwarden get pod vaultwarden-0 >/dev/null 2>&1; then
  kubectl exec -n vaultwarden vaultwarden-0 -- \
    sqlite3 /data/db.sqlite3 ".backup '/tmp/vaultwarden.db'"
  kubectl cp vaultwarden/vaultwarden-0:/tmp/vaultwarden.db "$BACKUP_TMP/vaultwarden.db"
fi

# Nextcloud (Postgres)
if kubectl -n nextcloud get pod nextcloud-postgresql-0 >/dev/null 2>&1; then
  kubectl exec -n nextcloud nextcloud-postgresql-0 -- \
    pg_dumpall -U postgres > "$BACKUP_TMP/nextcloud.sql"
fi

# Paperless (Postgres)
if kubectl -n paperless get pod paperless-db-0 >/dev/null 2>&1; then
  kubectl exec -n paperless paperless-db-0 -- \
    pg_dumpall -U paperless > "$BACKUP_TMP/paperless.sql"
fi

# Immich (Postgres on NAS via Docker)
if ssh nas "docker ps -q -f name=immich_postgres" | grep -q .; then
  ssh nas "docker exec immich_postgres pg_dumpall -U postgres" \
    > "$BACKUP_TMP/immich.sql"
fi

# --- Restic snapshot ---
restic backup "$BACKUP_TMP" --tag databases
restic backup /var/lib/rancher/k3s/server/manifests --tag manifests
restic backup /var/lib/rancher/k3s/server/db/snapshots --tag etcd
restic forget --keep-daily 14 --keep-weekly 8 --keep-monthly 12 --prune
rm -rf "$BACKUP_TMP"
restic check
```

## Schedule via Systemd Timer

`/etc/systemd/system/homelab-backup.timer`:

```ini
[Unit]
Description=Daily homelab backup

[Timer]
OnCalendar=*-*-* 03:00:00
RandomizedDelaySec=600
Persistent=true

[Install]
WantedBy=timers.target
```

Then enable: `sudo systemctl enable --now homelab-backup.timer`

!!! warning
    NAS snapshots are NOT backups — they live on the same disk pool. You need snapshots AND restic on a separate target.

## Test Your Restores

Once a month, restore latest snapshot to a temp directory and verify the DB opens.

!!! tip "Schedule a restore drill"
    The first time you discover backups are corrupt should NOT be when you need them.

## Verification

- [ ] Restic repo has snapshots:

    ```bash
    restic -r /mnt/backups/restic snapshots
    # Expected: at least one row with today's date
    ```

- [ ] Restore test (do this monthly):

    ```bash
    restic -r /mnt/backups/restic restore latest \
      --target /tmp/restore-test --tag databases
    # Then: sqlite3 /tmp/restore-test/path/to/db.sqlite3 .schema
    # (or pg_restore for postgres dumps) - just confirm files open
    ```

- [ ] Backup systemd timer is active:

    ```bash
    systemctl list-timers | grep homelab-backup
    ```

- [ ] Velero backup completed (if installed):

    ```bash
    velero backup get
    # Expected: STATUS=Completed for your latest backup
    ```
