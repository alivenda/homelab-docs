# Backups & DR

Local backups with a clearly-marked offsite TODO.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 2–4 hours |
| **Runs On** | NAS (primary backup target), cluster nodes |

## What runs when

Every job lands in its own bucket on the [Garage S3 store](#s3-object-storage-on-the-nas-garage) on the NAS, each with its own least-privilege key:

| Job | Schedule | What → bucket | Defined in |
|---|---|---|---|
| Velero `daily-cluster` | 04:00 UTC daily (7-day TTL) | every PVC (kopia filesystem backup) → `velero` | `homelab-manifests` — `infrastructure/velero/` |
| etcd snapshots | every 12 h | k3s etcd → local on ruby **and** `etcd-snapshots` | `homelab-ansible` — k3s `config.yaml` |
| Sealed-secrets key dump | 03:30 daily | all controller signing keys (rclone `crypt`) → `sealed-secrets-keys` | `homelab-manifests` — `infrastructure/sealed-secrets/` |
| Postgres dumps | 04:30 nightly | per-DB `pg_dump -Fc` + globals → `postgres-backups` | NAS timer — drill in [NAS PostgreSQL](nas-postgres.md) |
| Immich DB sync | 03:00 daily | Immich's own DB dumps → `immich-backups` | NAS timer — [below](#immich-database-garage) |
| Home Assistant sync | 05:30 daily | HA's native backups (SMB pull) → `ha-backups` | NAS timer — [below](#home-assistant-backups-garage) |
| Audiobookshelf sync | 06:00 daily | ABS's native backups → `audiobookshelf-backups` | NAS timer — [below](#audiobookshelf-config-garage) |

!!! warning "Off-site backup is still a TODO"
    Everything above lands on the NAS — that protects against disk failure but NOT fire/theft/flood. Options: friend's house with Tailscale + Restic, cycled external drives, second location you control. Schedule a calendar reminder within 3 months.

## Strategy: the tiers

| Tier | What | Where |
|---|---|---|
| Tier 1 — Critical | Vaultwarden, Immich DB, Nextcloud DB, Paperless DB, k3s manifests | NAS daily + retained 90d |
| Tier 2 — Important | Service configs, ArgoCD state, Grafana dashboards | NAS daily + retained 30d |
| Tier 3 — Replaceable | Container images, media | NAS weekly + retained 14d |

The tiers are the policy; [What runs when](#what-runs-when) is the implementation. The tiers cover application data (DB dumps) and IaC (manifests) but not Kubernetes objects themselves — PVCs, CRDs, secrets in non-Git-tracked namespaces, helm release state. That gap is what [Velero](#velero-for-k8s-native-pvc-backup) closes.

## Secrets and key-material recovery

This is **step zero of any real disaster recovery**. Velero and the database-dump jobs restore your *data*; this section restores the ability to *decrypt* it. After rebuilding a machine or the cluster, do this first — nothing else works until it's done.

### The single root of trust

Every secret in the homelab funnels through **one age keypair**:

- The same recipient — `age164pxwzqulte2t6uh6vpkg4kd84uvk0cks5gzg3wc508lvs0x7syskmykd9` — encrypts every SOPS file across `homelab-ansible`, `homelab-terraform`, and `homelab-secrets`.
- The private key lives at `~/.config/sops/age/keys.txt`. Its only off-machine copy is a secure-note item in your **externally-hosted password manager** (a hosted service — deliberately not anything this homelab runs).
- The Sealed Secrets controller's signing-key backup (`homelab-secrets/sealed-secrets-controller-key.enc.yaml`) is itself SOPS/age-encrypted — so it, too, is locked behind that one age key.

Recovery therefore runs in a strict order, rooted on that external password manager:

```
External password manager ──> age private key ──┬──> SOPS files (Ansible + Terraform secrets)
                                                └──> Sealed Secrets signing key ──> cluster SealedSecrets
```

!!! danger "The external password manager is the keystone — make sure *it* is independently recoverable"
    Everything below decrypts from one age key whose only off-machine copy is in that manager. If you can't get into it, nothing is recoverable. Confirm now: master password memorized (not stored only inside the vault), and its two-factor **recovery code** printed and kept offline (fireproof safe / second location). Note that **Vaultwarden runs inside this cluster** — never make the cluster's recovery depend on a secrets store the cluster itself hosts. The root of trust must be an externally-hosted manager (or an offline vault export), not Vaultwarden.

### Step 1 — Restore the age private key

On your machine (or any host that will run `sops`/`tofu`/`ansible`):

```sh
mkdir -p ~/.config/sops/age
```

Open your password manager and retrieve the saved homelab age key — the secure note holding the `AGE-SECRET-KEY-1…` line plus its `# public key:` comment — and save it into `~/.config/sops/age/keys.txt`. Pasting into an editor avoids any shell-quoting pitfalls. Then lock the file down:

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

!!! warning "Restore ALL keys, not just the day-zero one"
    The controller rotates keys every 30 days, and each SealedSecret decrypts only under
    the key it was sealed with. The authoritative source is the **automated Garage dump**
    (all keys — restore drill in `homelab-manifests/infrastructure/sealed-secrets/README.md`);
    the `homelab-secrets` file below is the day-zero fallback and only covers secrets
    sealed before the first rotation.

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

The controller mints a **new key every 30 days** and keeps the old ones — every key ever created stays required, because each SealedSecret decrypts only under the key it was sealed with. A one-time export therefore goes stale at the first rotation.

This build automates it: the daily in-cluster **CronJob dumps all controller keys to a dedicated Garage bucket** through an rclone `crypt` remote, so the bucket only ever holds ciphertext. The manifests, the crypt-password handling, and the restore drill live in `homelab-manifests/infrastructure/sealed-secrets/` — that dump is the authoritative ongoing backup.

The `homelab-secrets` copy (`sealed-secrets-controller-key.enc.yaml`) is the **day-zero export**: the fallback if Garage itself is lost, covering only SealedSecrets sealed before the first rotation. Re-taking it manually is optional; if you do, this is the shape:

```sh
kubectl get secret -n sealed-secrets \
  -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > controller-key.yaml
sops --encrypt controller-key.yaml > sealed-secrets-controller-key.enc.yaml
rm controller-key.yaml   # never commit the plaintext
```

## S3 object storage on the NAS (Garage)

This S3-compatible store on the NAS backs every job in [What runs when](#what-runs-when), at `http://10.0.20.50:9000`. Each consumer — etcd, Velero, the Terraform state backend, the NAS-side sync jobs — gets its own bucket and its own scoped key.

!!! warning "Why Garage, not MinIO"
    MinIO's Community Edition is effectively end-of-life: the `minio/minio` repo was archived (read-only) in **April 2026**, free Docker/Quay image publishing stopped in **October 2025** (last tag `RELEASE.2025-10-15T17-29-55Z`), and the admin console was stripped from CE in **May 2025**. This runbook uses [Garage](https://garagehq.deuxfleurs.fr/) — a maintained, Rust, self-host-focused S3 store — instead.

### Stand up Garage

Garage's S3 API is plain HTTP, so clients connect with `etcd-s3-insecure` / `s3ForcePathStyle`. Binding on `:9000` with `s3_region = "us-east-1"` matches k3s's default `--etcd-s3-region`, so the Ansible etcd config needs no endpoint/region overrides.

??? example "`/volume1/docker/garage/garage.toml`"

    Generate the two secrets with `openssl rand -hex 32` and `openssl rand -base64 32`:

    ```toml
    replication_factor = 1
    metadata_dir = "/var/lib/garage/meta"
    data_dir     = "/var/lib/garage/data"
    db_engine    = "lmdb"

    rpc_bind_addr   = "[::]:3901"
    rpc_public_addr = "127.0.0.1:3901"   # single-node self-reference
    rpc_secret      = "<openssl rand -hex 32>"

    [s3_api]
    api_bind_addr = "[::]:9000"   # bind S3 on :9000 …
    s3_region     = "us-east-1"   # … with region us-east-1 to match k3s's default

    [admin]
    api_bind_addr = "127.0.0.1:3903"
    admin_token   = "<openssl rand -base64 32>"
    ```

??? example "`/volume1/docker/garage/docker-compose.yml`"

    ```yaml
    services:
      garage:
        image: dxflrs/garage:v2.3.0
        container_name: garage
        command: ["/garage", "server"]
        volumes:
          - /volume1/docker/garage/garage.toml:/etc/garage.toml:ro
          - /volume1/docker/garage/meta:/var/lib/garage/meta
          - /volume1/docker/garage/data:/var/lib/garage/data
        ports:
          - "9000:9000"   # S3 API
        restart: unless-stopped
    ```

```bash
mkdir -p /volume1/docker/garage/meta /volume1/docker/garage/data
docker compose -f /volume1/docker/garage/docker-compose.yml up -d
docker logs garage   # confirm: "S3 API server listening on http://[::]:9000"
```

### Initialize the cluster + per-consumer keys

A one-time single-node layout, then a dedicated bucket and least-privilege key per consumer:

```bash
docker exec -ti garage /garage status                               # copy the node ID
docker exec -ti garage /garage layout assign -z nas -c 10G <NODE_ID>
docker exec -ti garage /garage layout apply --version 1

# One bucket + scoped key per consumer (example: etcd snapshots)
docker exec -ti garage /garage bucket create etcd-snapshots
docker exec -ti garage /garage key create etcd-backup               # copy the Key ID (GK…) and Secret
docker exec -ti garage /garage bucket allow --read --write etcd-snapshots --key etcd-backup
```

Repeat the bucket/key/allow trio for `velero` (and any other consumer) so each gets its own credentials.

### Off-node etcd snapshots (k3s-native)

k3s uploads scheduled etcd snapshots straight to Garage — no extra tooling. It's codified in `homelab-ansible` (`site.yml` renders `/etc/rancher/k3s/config.yaml`); the access/secret key live in that repo's SOPS file as `etcd_s3_access_key` / `etcd_s3_secret_key`:

```yaml
etcd-snapshot-schedule-cron: "0 */12 * * *"
etcd-snapshot-retention: 10
etcd-s3: true
etcd-s3-endpoint: "10.0.20.50:9000"
etcd-s3-insecure: true          # Garage serves plain HTTP on :9000
etcd-s3-bucket: "etcd-snapshots"
etcd-s3-access-key: "<from SOPS>"
etcd-s3-secret-key: "<from SOPS>"
```

k3s keeps **both** copies — local (`file://`) and off-node (`s3://`) — with retention applied to each. The save succeeds locally even if the S3 upload fails, so confirm the off-node copy explicitly:

```bash
sudo k3s etcd-snapshot save
sudo k3s etcd-snapshot list                               # expect an s3://etcd-snapshots/… row
docker exec -ti garage /garage bucket info etcd-snapshots # expect Objects ≥ 1
```

This is the primary off-node etcd path (k3s also keeps local snapshots on ruby).

## Velero for k8s-native PVC backup

Velero's filesystem backup (the **node-agent**, using the kopia uploader — the default since Velero 1.10) snapshots PVC contents and stores them in the Garage store above, in a dedicated `velero` bucket.

!!! note "GitOps-managed in this build"
    Velero runs as two ArgoCD Applications (`bootstrap/velero.yaml` — chart + manifests,
    the same split Forgejo and Woodpecker use), with live values in
    `infrastructure/velero/values.yaml`. The values below are the reference for *what*
    the settings mean; the collapsed block holds the imperative bootstrap for a
    pre-GitOps install.

The Velero chart's schema changed in v3.0.0+: `backupStorageLocation` is now an array, `provider` lives inside each entry, and the old `deployRestic` flag was renamed `deployNodeAgent`.

`velero-values.yaml`:

```yaml
configuration:
  backupStorageLocation:
    - name: default
      provider: aws       # use 'aws' provider for any S3-compatible target
      bucket: velero
      default: true
      config:
        region: us-east-1               # matches Garage's s3_region
        s3Url: http://10.0.20.50:9000   # NAS (Networking static-IP table)
        s3ForcePathStyle: "true"        # Garage uses path-style addressing

deployNodeAgent: true       # replaces deployRestic; needed for PVC-content backups

credentials:
  useSecret: true
  # secretContents.cloud comes in via --set-file at install time
```

??? example "Imperative bootstrap (credentials file + helm install)"

    ```bash
    # On your machine (wherever you'll run the helm commands)
    cat > velero-creds <<EOF
    [default]
    aws_access_key_id = <GARAGE_VELERO_KEY_ID>
    aws_secret_access_key = <GARAGE_VELERO_SECRET>
    EOF

    helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts

    helm upgrade --install velero vmware-tanzu/velero \
      --version <X.Y.Z> \
      --namespace velero --create-namespace \
      --values velero-values.yaml \
      --set-file credentials.secretContents.cloud=./velero-creds
    ```

    Pin `--version` to a current release listed on [vmware-tanzu/helm-charts](https://github.com/vmware-tanzu/helm-charts/tree/main/charts/velero).

```bash
# Verify
velero backup-location get

# Take a full cluster backup
velero backup create homelab-$(date +%Y%m%d)
```

!!! tip "Off-site backup target"
    Velero (and the etcd S3 upload) can point at any S3-compatible target, not just the NAS — Backblaze B2, Wasabi, or Cloudflare R2 for off-site storage. R2's free tier is generous for homelab volumes — point the same config at R2 and you get cloud-hosted backups for free.

## Relational database dumps → Garage

The relational database tier lives on the NAS, not the cluster (see the [Storage & Data Architecture](storage-architecture.md)), so database backups run **on the NAS**, next to the server: nightly per-database `pg_dump -Fc` plus a `pg_dumpall --globals-only`, pushed to a dedicated `postgres-backups` Garage bucket by a `postgres-backup.timer` (04:30, an hour before the HA sync so the jobs don't contend for NAS I/O). The full pipeline — script, units, and the seeded restore drill that gates it — is in [NAS PostgreSQL](nas-postgres.md).

Two things stay out of this shared job, each for its own reason:

- **Immich** keeps its own dump path — its bundled Postgres on the NAS predates the shared server ([Immich](immich.md)), and Immich's built-in scheduled backup already dumps the database itself. Getting those dumps **off-box** is its own sync job — see [Immich database → Garage](#immich-database-garage) below.
- **Embedded-SQLite apps** — their volumes live on `local-path`, which Velero's node-agent already captures above. One data tier, one backup mechanism; no double-coverage. (Audiobookshelf used to be in this set; it moved to the NAS and now has its own job — see below.)

## NAS-side sync jobs: the rclone → Garage pattern

Three apps run on NAS-adjacent hosts outside both Velero's and the Postgres job's reach. Each follows the same shape: **the app writes its own consistent backups locally, and a NAS systemd timer ships them to a per-app Garage bucket with rclone, pruning the bucket to a rolling 30-day window.** Letting each app produce the backup itself avoids copying live databases mid-write.

| Job | App writes to | Timer | Bucket | rclone remote |
|---|---|---|---|---|
| [Immich](#immich-database-garage) | `/volume1/photos/backups` (NAS-local) | 03:00 | `immich-backups` | `[garage-immich]` |
| [Home Assistant](#home-assistant-backups-garage) | HA `/backup` on slate — pulled over SMB | 05:30 | `ha-backups` | `[garage]` + `[hasmb]` |
| [Audiobookshelf](#audiobookshelf-config-garage) | `/volume1/docker/audiobookshelf/metadata/backups` (NAS-local) | 06:00 | `audiobookshelf-backups` | `[garage-abs]` |

!!! warning "Copy, never sync — and mind UGOS updates"
    Use `rclone copy` (plus the age-based `delete`), never `rclone sync`: `sync` mirrors deletions, so if the app's backup directory is ever empty (reinstall, disk loss) it would wipe the Garage copy too. And because the NAS runs **UGOS Pro** (an appliance OS), these host units live outside Ansible's reach — a major UGOS update can reset them, so the sections below are the recovery reference.

Each job gets the standard Garage consumer (the bucket/key/allow trio from [above](#initialize-the-cluster-per-consumer-keys)) and an S3 remote of the same shape in the NAS's `/etc/rclone/rclone.conf` (root-owned, `chmod 600`):

```ini
[garage-<consumer>]
type = s3
provider = Other
endpoint = http://10.0.20.50:9000
region = us-east-1          # must match garage.toml's s3_region
access_key_id = GK…
secret_access_key = <secret>
force_path_style = true     # Garage requires path-style
```

(The Home Assistant job predates the naming convention and calls its S3 remote plain `[garage]`; it also needs the extra `[hasmb]` SMB remote — see its section.)

### Immich database → Garage

Immich [runs as Docker on the NAS](immich.md) with its **own** bundled Postgres, separate from the shared server above. It already backs that database up on a schedule — Immich's built-in job writes a version-stamped dump to `${UPLOAD_LOCATION}/backups/` (e.g. `/volume1/photos/backups/immich-db-backup-20260617T020000-v2.7.5-pg14.19.sql.gz`) nightly at 02:00. What that leaves open is **off-box** durability: those dumps land on the same volume as the photo library, so one volume failure loses the originals *and* their database together, and the [cold-shutdown export](cold-shutdown.md) only sweeps Garage buckets.

First confirm Immich's built-in backup is on: **Administration → Settings → Backup Settings → Database Backups** (enabled by default). The library/originals themselves are the bulk data on `${UPLOAD_LOCATION}` — re-uploadable from your devices, not part of this database job.

Garage consumer + `[garage-immich]` remote per the pattern above. The dumps are a local NAS directory, so rclone copies straight off disk.

??? example "`immich-backup-sync` service + timer"

    `/etc/systemd/system/immich-backup-sync.service`:

    ```ini
    [Unit]
    Description=Sync Immich database backups to Garage
    After=network-online.target
    Wants=network-online.target

    [Service]
    Type=oneshot
    ExecStart=/usr/bin/docker run --rm --user root -v /etc/rclone:/config/rclone -v /volume1/photos/backups:/immich-backups:ro rclone/rclone copy /immich-backups garage-immich:immich-backups -v
    ExecStart=/usr/bin/docker run --rm --user root -v /etc/rclone:/config/rclone rclone/rclone delete garage-immich:immich-backups --min-age 30d
    ```

    `/etc/systemd/system/immich-backup-sync.timer`:

    ```ini
    [Unit]
    Description=Daily Immich backup sync to Garage

    [Timer]
    OnCalendar=*-*-* 03:00:00      # after Immich's 02:00 database-backup window
    Persistent=true

    [Install]
    WantedBy=timers.target
    ```

```bash
systemctl daemon-reload && systemctl enable --now immich-backup-sync.timer
systemctl list-timers immich-backup-sync.timer
docker exec -ti garage /garage bucket info immich-backups          # Objects ≥ 1 after the first run
```

### Home Assistant backups → Garage

Home Assistant runs off-cluster — an HAOS VM on **slate** (`10.0.20.21`, see [Home Assistant](home-assistant.md)). Its native backups are local; getting them off-box to Garage is done by **pulling from the NAS with rclone**, not by an in-HA S3 integration.

!!! note "Why pull from the NAS instead of an HA S3 backup-agent integration"
    Home Assistant's S3-compatible backup-agent integrations are `botocore`-based, and on current HA they break on an `aiobotocore`↔`botocore` version skew (the integration's newer `aiobotocore` passes an argument the bundled `botocore` doesn't accept). Decoupling — HA writes local backups, the NAS ships them to Garage — sidesteps HA's Python entirely and survives HA core updates, which is the better DR posture regardless.

On the HA side:

1. Install the official **Samba share** add-on; set a username + password and scope **Allowed Hosts** to the Lab VLAN (`10.0.20.0/24`) — only it should reach the shares (this exposes `/config` too). HA's `/backup` is then readable as the `backup` share.
2. Settings → System → **Backups** → automatic backup: daily, keep 7. These write to `/backup`.
3. **Store the backup encryption key in Vaultwarden** (and on paper), alongside the age key. HA encrypts every backup; without the key the Garage copy is unrecoverable.

On the NAS side, this job needs an SMB remote in addition to its S3 one — obscure the Samba password with `docker run --rm rclone/rclone obscure '<password>'`:

```ini
[hasmb]
type = smb
host = 10.0.20.21
user = <ha-samba-user>
pass = <obscured>
```

Test the connection, then copy:

```bash
docker run --rm --user root -v /etc/rclone:/config/rclone rclone/rclone lsd hasmb:           # lists shares → SMB auth works
docker run --rm --user root -v /etc/rclone:/config/rclone rclone/rclone copy hasmb:backup garage:ha-backups -v
docker exec -ti garage /garage bucket info ha-backups                                        # Objects ≥ 1
```

??? example "`ha-backup-sync` service + timer"

    `/etc/systemd/system/ha-backup-sync.service`:

    ```ini
    [Unit]
    Description=Sync Home Assistant backups to Garage
    After=network-online.target
    Wants=network-online.target

    [Service]
    Type=oneshot
    ExecStart=/usr/bin/docker run --rm --user root -v /etc/rclone:/config/rclone rclone/rclone copy hasmb:backup garage:ha-backups -v
    ExecStart=/usr/bin/docker run --rm --user root -v /etc/rclone:/config/rclone rclone/rclone delete garage:ha-backups --min-age 30d
    ```

    `/etc/systemd/system/ha-backup-sync.timer`:

    ```ini
    [Unit]
    Description=Daily HA backup sync to Garage

    [Timer]
    OnCalendar=*-*-* 05:30:00      # after HA's automatic-backup window
    Persistent=true

    [Install]
    WantedBy=timers.target
    ```

```bash
systemctl daemon-reload && systemctl enable --now ha-backup-sync.timer
systemctl list-timers ha-backup-sync.timer     # confirm a NEXT run
```

### Audiobookshelf config → Garage

Audiobookshelf [runs as a Docker container on the NAS](apps-catalog.md#audiobookshelf), so — unlike the embedded-SQLite apps in the cluster — its `/config` database is **not** on `local-path` and is **not** covered by Velero.

In ABS **Settings → Backups**, enable scheduled backups (daily; keep ~7). ABS writes a consistent archive to `BACKUP_PATH` (default `/metadata/backups`, i.e. `/volume1/docker/audiobookshelf/metadata/backups`) containing the **`/config` database** (users, libraries, OIDC config, the mobile-redirect whitelist) plus item/author images from `/metadata`. The library audio itself is **not** included — that's the original media on `/volume1/media`, re-scannable.

Garage consumer + `[garage-abs]` remote per the pattern above. The backups are a local NAS directory, so rclone copies straight off disk.

??? example "`audiobookshelf-backup-sync` service + timer"

    `/etc/systemd/system/audiobookshelf-backup-sync.service`:

    ```ini
    [Unit]
    Description=Sync Audiobookshelf backups to Garage
    After=network-online.target
    Wants=network-online.target

    [Service]
    Type=oneshot
    ExecStart=/usr/bin/docker run --rm --user root -v /etc/rclone:/config/rclone -v /volume1/docker/audiobookshelf/metadata/backups:/abs-backups:ro rclone/rclone copy /abs-backups garage-abs:audiobookshelf-backups -v
    ExecStart=/usr/bin/docker run --rm --user root -v /etc/rclone:/config/rclone rclone/rclone delete garage-abs:audiobookshelf-backups --min-age 30d
    ```

    `/etc/systemd/system/audiobookshelf-backup-sync.timer`:

    ```ini
    [Unit]
    Description=Daily Audiobookshelf backup sync to Garage

    [Timer]
    OnCalendar=*-*-* 06:00:00      # after ABS's own backup window
    Persistent=true

    [Install]
    WantedBy=timers.target
    ```

```bash
systemctl daemon-reload && systemctl enable --now audiobookshelf-backup-sync.timer
systemctl list-timers audiobookshelf-backup-sync.timer
docker exec -ti garage /garage bucket info audiobookshelf-backups     # Objects ≥ 1 after the first run
```

## Test your restores

A backup that has never been restored is a hypothesis, not a backup. Once a month, restore something real and check **content**, not exit codes:

- A Velero `PodVolumeBackup` must show non-zero bytes — `Completed` alone proves nothing (see the [storage architecture](storage-architecture.md#the-local-path-tier) for how that failure mode was caught).
- A database dump must restore from the **Garage copy** into a scratch database with matching row counts — [NAS PostgreSQL](nas-postgres.md) has the drill.

!!! tip "Schedule a restore drill"
    The first time you discover backups are corrupt should NOT be when you need them.

## Verification

- [ ] The external password manager's two-factor **recovery code** is printed and stored offline (the keystone — see [The single root of trust](#the-single-root-of-trust)).
- [ ] age key restores from the external password manager and derives the expected public key:

    ```sh
    age-keygen -y ~/.config/sops/age/keys.txt
    # Expected: age164pxwzqulte2t6uh6vpkg4kd84uvk0cks5gzg3wc508lvs0x7syskmykd9
    ```

- [ ] `sops --decrypt` succeeds on a known encrypted file (e.g. `homelab-secrets/sealed-secrets-controller-key.enc.yaml`).
- [ ] Database backup timer is active on the NAS, and the bucket holds real objects:

    ```bash
    systemctl list-timers postgres-backup.timer            # a NEXT run is scheduled
    docker exec -ti garage /garage bucket info postgres-backups
    # Expected: Objects ≥ 1 with non-trivial sizes
    ```

- [ ] Restore test (do this monthly): restore a dump from the Garage copy into a scratch
  database and compare row counts against the source — the drill in
  [NAS PostgreSQL](nas-postgres.md) is the template.

- [ ] Off-node etcd snapshot present in Garage:

    ```bash
    docker exec -ti garage /garage bucket info etcd-snapshots
    # Expected: Objects ≥ 1
    ```

- [ ] Velero backup completed (if installed):

    ```bash
    velero backup get
    # Expected: STATUS=Completed for your latest backup
    ```
