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

## Secrets and key-material recovery

This is **step zero of any real disaster recovery**. Restic and Velero restore your *data*; this section restores the ability to *decrypt* it. After rebuilding a machine or the cluster, do this first — nothing else works until it's done.

### The single root of trust

Every secret in the homelab funnels through **one age keypair**:

- The same recipient — `age164pxwzqulte2t6uh6vpkg4kd84uvk0cks5gzg3wc508lvs0x7syskmykd9` — encrypts every SOPS file across `homelab-ansible`, `homelab-terraform`, and `homelab-secrets`.
- The private key lives at `~/.config/sops/age/keys.txt`. Its only off-machine copy is the Bitwarden item **"sops homelab decryption key"**.
- The Sealed Secrets controller's signing-key backup (`homelab-secrets/sealed-secrets-controller-key.enc.yaml`) is itself SOPS/age-encrypted — so it, too, is locked behind that one age key.

Recovery therefore runs in a strict order, rooted on Bitwarden:

```
Bitwarden ──> age private key ──┬──> SOPS files (Ansible + Terraform secrets)
                                └──> Sealed Secrets signing key ──> cluster SealedSecrets
```

!!! danger "Bitwarden is the keystone — make sure *it* is independently recoverable"
    Everything below decrypts from one age key whose only off-machine copy is in Bitwarden. If you can't get into Bitwarden, nothing is recoverable. Confirm now: master password memorized (not stored only inside the vault), and the Bitwarden two-factor **recovery code** printed and kept offline (fireproof safe / second location). Note that **Vaultwarden runs inside this cluster** — never make the cluster's recovery depend on a secrets store the cluster itself hosts. The root of trust must be the externally-hosted Bitwarden (or an offline `bw export`), not Vaultwarden.

### Step 1 — Restore the age private key

On your machine (or any host that will run `sops`/`tofu`/`ansible`):

```sh
mkdir -p ~/.config/sops/age
```

Open the Bitwarden item **"sops homelab decryption key"**, copy its contents (the `AGE-SECRET-KEY-1…` line plus the `# public key:` comment), and save them into `~/.config/sops/age/keys.txt`. Pasting into an editor avoids any shell-quoting pitfalls. Then lock the file down:

```sh
chmod 600 ~/.config/sops/age/keys.txt
```

Verify it is the correct key — the derived public key must equal the recipient in every repo's `.sops.yaml`:

```sh
age-keygen -y ~/.config/sops/age/keys.txt
# Expected: age164pxwzqulte2t6uh6vpkg4kd84uvk0cks5gzg3wc508lvs0x7syskmykd9
```

If that matches, the repo-side secret layer is recoverable. SOPS reads `~/.config/sops/age/keys.txt` automatically; if you keep the key elsewhere, point `SOPS_AGE_KEY_FILE` at it.

### Step 2 — Confirm you can decrypt the repos

```sh
sops --decrypt homelab-secrets/sealed-secrets-controller-key.enc.yaml | head
```

A successful decrypt confirms the whole SOPS layer. Each repo unlocks a different part of the rebuild:

| Repo | File | Unlocks |
|---|---|---|
| `homelab-ansible` | `secrets/secrets.sops.yaml` | `k3s_token`, `tailscale_authkey` — needed to re-bootstrap the cluster and Tailscale routers |
| `homelab-terraform` | `cloudflare/secrets.enc.yaml` | Cloudflare API token — needed for `tofu apply` |
| `homelab-secrets` | `sealed-secrets-controller-key.enc.yaml` | the cluster's Sealed Secrets signing key (Step 3) |

### Step 3 — Restore the Sealed Secrets signing key (cluster rebuild only)

Only needed when the cluster was rebuilt. A fresh Sealed Secrets controller generates a **new** keypair and cannot decrypt secrets that were sealed against the old one — so every `SealedSecret` committed to `homelab-manifests` would be undecryptable. Restoring the backed-up signing key avoids re-sealing anything.

```bash
# 1. Ensure the controller exists (ArgoCD installs it into the sealed-secrets namespace).
kubectl get deploy -n sealed-secrets sealed-secrets-controller

# 2. Decrypt the backed-up signing key on your machine and apply it to the cluster.
sops --decrypt sealed-secrets-controller-key.enc.yaml | kubectl apply -f -

# 3. Restart the controller so it loads the restored key.
kubectl delete pod -n sealed-secrets -l app.kubernetes.io/name=sealed-secrets

# 4. Confirm the restored key is present and active.
kubectl get secret -n sealed-secrets -l sealedsecrets.bitnami.com/sealed-secrets-key
# Expected: a kubernetes.io/tls secret labelled ...sealed-secrets-key=active
```

Re-syncing `homelab-manifests` in ArgoCD will now decrypt every existing SealedSecret normally — no manifest changes required.

### Keep the signing-key backup current

The controller rotates/renews its key on a schedule and on some upgrades, so the backup goes stale. Re-take it after any Sealed Secrets controller upgrade or key rotation, then commit via a branch + PR to `homelab-secrets`:

```sh
kubectl get secret -n sealed-secrets \
  -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > controller-key.yaml
sops --encrypt controller-key.yaml > sealed-secrets-controller-key.enc.yaml
rm controller-key.yaml   # never commit the plaintext
```

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

Run on ruby (where `kubectl` works against the cluster). Comment out the lines for services you haven't deployed yet — the `kubectl get pod` guard around each block skips a service whose pod is absent, so commenting out keeps the intent explicit.

!!! warning "Verify the pod names against your charts"
    The guards below hard-code StatefulSet-style names (`vaultwarden-0`, `nextcloud-postgresql-0`, `paperless-db-0`). If a chart version deploys the DB as a Deployment or under a different release name, the guard silently evaluates false and that database is **skipped without error** — the most dangerous backup failure mode. Confirm each name with `kubectl get pods -n <namespace>` before trusting the script, and prefer a label selector (e.g. `kubectl get pod -n nextcloud -l app.kubernetes.io/name=postgresql -o name`) if your chart's pod names aren't stable.

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

- [ ] Bitwarden two-factor **recovery code** is printed and stored offline (the keystone — see [The single root of trust](#the-single-root-of-trust)).
- [ ] age key restores from Bitwarden and derives the expected public key:

    ```sh
    age-keygen -y ~/.config/sops/age/keys.txt
    # Expected: age164pxwzqulte2t6uh6vpkg4kd84uvk0cks5gzg3wc508lvs0x7syskmykd9
    ```

- [ ] `sops --decrypt` succeeds on a known encrypted file (e.g. `homelab-secrets/sealed-secrets-controller-key.enc.yaml`).
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
