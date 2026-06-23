# NAS PostgreSQL

One PostgreSQL server on the NAS, one database + role per app. This is the "Relational DB"
tier from the [Storage & Data Architecture](storage-architecture.md) — the cluster runs app
pods, the NAS runs their databases.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 1–2 hours |
| **Runs On** | NAS (Docker — not k3s) |
| **Depends On** | Backups (Garage S3), [Storage & Data Architecture](storage-architecture.md) |

## Does your app even belong here?

The architecture splits database-like data two ways, and sending an app to the wrong tier
undoes the design:

- **Embedded SQLite** (Vaultwarden, Forgejo, Actual Budget, the Arr stack, …)
  → stays **in-cluster** on `local-path` + `strategy: Recreate`. SQLite is a library inside
  the app process, not a server — there is nothing to centralize, and putting its file on
  network storage is the corruption pattern this architecture exists to prevent. These apps
  never touch this server.
- **Client/server relational databases** (Nextcloud, Paperless-ngx, Vikunja → PostgreSQL;
  BookStack → MariaDB) → a database + role **here**, connected over the Lab VLAN.

If the app's docs offer both ("SQLite by default, Postgres supported"), prefer SQLite on
`local-path` for small single-user apps and Postgres here for anything multi-user or
write-heavy — the [decomposition rule](storage-architecture.md#how-to-decompose-one-app)
is the tie-breaker.

## Why port 5433

UGOS Pro (the NAS appliance OS) runs its **own internal PostgreSQL** on `127.0.0.1:5432`
for its services. A loopback bind wouldn't technically collide with a bind on the LAN IP,
but a firmware update could change it, and the standing rule for the appliance is *don't
fight UGOS's config layer*. So this server publishes **`10.0.20.50:5433`** — every consumer
`DATABASE_URL` must say port `5433`.

Check before you build (also confirms Immich's bundled Postgres stays unpublished, and that
you have disk headroom):

```bash
sudo docker ps --format '{{.Names}}  {{.Ports}}'
sudo ss -tlnp | grep 5432
df -h /volume1
```

## Stand up the server

Create `/volume1/docker/postgres/` with three files.

`docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:18
    container_name: postgres
    restart: unless-stopped
    # Postgres uses shared memory for parallel work; docker's 64MB default
    # is a known source of spurious query failures.
    shm_size: 256mb
    ports:
      # Lab-VLAN IP only — never 0.0.0.0. And 5433: see above.
      - "10.0.20.50:5433:5432"
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    # Replace the image's default allow-all pg_hba with ours.
    command: postgres -c hba_file=/etc/postgresql/pg_hba.conf
    volumes:
      - ./data:/var/lib/postgresql
      - ./pg_hba.conf:/etc/postgresql/pg_hba.conf:ro
```

!!! danger "postgres:18 changed the volume path"
    From PostgreSQL 18 the image's volume is **`/var/lib/postgresql`** (PGDATA moved to
    `/var/lib/postgresql/18/docker`, per-major, to enable `pg_upgrade --link`). The familiar
    pre-18 mount at `/var/lib/postgresql/data` no longer matches the declared volume — data
    lands in an anonymous volume and **vanishes when the container is re-created**.

`pg_hba.conf`:

```
# Admin path is the local socket via docker exec — no superuser over TCP.
local   all   all                  trust
# Loopback inside the container only.
host    all   all   127.0.0.1/32   scram-sha-256
host    all   all   ::1/128        scram-sha-256
# Per-app rules get appended at each app's bring-up (then reload, no restart):
#   host  <db>  <role>  10.0.20.10/32  scram-sha-256   (one line per node IP)
# Everything not matched above is implicitly rejected.
```

!!! warning "The hba file must be readable by uid 999"
    Postgres runs in-container as uid 999. A root-owned `pg_hba.conf` created with a
    restrictive mode crash-loops the server (`could not open file … Permission denied` →
    `FATAL`). `chmod 644` it — it contains policy, not secrets. The `.env` below is the
    file that stays `600`.

`.env` — generate the superuser password, store it in your password manager first, then:

```
POSTGRES_PASSWORD=<generated password>
```

```bash
openssl rand -base64 24          # → password manager, then .env
sudo chmod 644 pg_hba.conf
sudo chmod 600 .env
sudo docker compose up -d
```

Verify:

```bash
sudo docker logs postgres --tail 15
sudo docker exec -ti postgres psql -U postgres -c 'select version();'
sudo docker exec -ti postgres psql -U postgres -c 'show data_checksums;'
sudo ss -tlnp | grep 5433
```

Expected: logs end `database system is ready to accept connections` with no hba errors,
version 18.x, `data_checksums = on` (the PG 18 initdb default — page corruption gets
*detected* instead of silently served), and a listener on `10.0.20.50:5433` only.

!!! tip "If the very first boot failed"
    A failed first start can leave a half-initialized `./data` where the superuser password
    was never applied ("Database directory appears to contain a database; Skipping
    initialization"). Before any real data exists, the clean fix is `docker compose down`,
    fix the cause, `rm -rf data`, and let `initdb` run again.

## The cluster-side gate

From your machine, prove both reachability and lockdown in one command:

```bash
kubectl run pg-test --rm -it --restart=Never --image=postgres:18 \
  --env=PGPASSWORD=x --command -- psql -h 10.0.20.50 -p 5433 -U postgres -c 'select 1;'
```

**Pass is a rejection**:

```
FATAL:  no pg_hba.conf entry for host "10.0.20.10", user "postgres", database "postgres"
```

That single line proves the TCP path from cluster pods works, Postgres itself answered, and
the hba policy refuses anything not explicitly granted — before even asking for a password.
Note the host IP in the message: it's a **node** IP, because pod egress to the NAS is
SNAT'd. That's why the per-app hba rules below key on node IPs, not pod CIDRs.

## Provisioning a database for an app

At each relational app's bring-up (not in advance):

```bash
sudo docker exec -ti postgres psql -U postgres \
  -c "CREATE ROLE <app> LOGIN PASSWORD '<generated>'" \
  -c "CREATE DATABASE <app> OWNER <app>"
```

Append to `pg_hba.conf` — one line per node, scoped to exactly this database and role:

```
host  <app>  <app>  10.0.20.10/32  scram-sha-256
host  <app>  <app>  10.0.20.11/32  scram-sha-256
host  <app>  <app>  10.0.20.12/32  scram-sha-256
host  <app>  <app>  10.0.20.13/32  scram-sha-256
```

Then reload (no restart): `sudo docker exec -ti postgres psql -U postgres -c 'SELECT pg_reload_conf()'`

The app password goes to your password manager and into a `SealedSecret` in that app's
directory in `homelab-manifests`. The connection string the app consumes:

```
postgresql://<app>:<password>@10.0.20.50:5433/<app>
```

## Backups: nightly dumps → Garage

Same per-consumer pattern as every other Garage client (Backups): dedicated bucket +
least-privilege key, rclone remote, systemd timer on the NAS.

```bash
sudo docker exec -ti garage /garage bucket create postgres-backups
sudo docker exec -ti garage /garage key create postgres-backup     # copy Key ID + Secret
sudo docker exec -ti garage /garage bucket allow --read --write postgres-backups --key postgres-backup
```

Append to `/etc/rclone/rclone.conf` (the HA remote keeps its own key — per-consumer creds):

```ini
[garage-pg]
type = s3
provider = Other
endpoint = http://10.0.20.50:9000
region = us-east-1
access_key_id = GK…
secret_access_key = <secret>
force_path_style = true
```

`/volume1/docker/postgres/backup.sh` (`chmod 755`):

```bash
#!/bin/bash
# Nightly logical backups of the shared Postgres to Garage S3.
set -euo pipefail

STAMP=$(date +%F)
TMP=$(mktemp -d)
trap 'rm -rf "$TMP"' EXIT

# Roles and grants live outside any single database — dump them separately,
# or a restore onto a fresh server has databases but no logins.
docker exec postgres pg_dumpall -U postgres --globals-only > "$TMP/globals-$STAMP.sql"

# Each real database individually, custom format: pg_restore can then do
# selective, per-app restores — pg_dumpall's plain SQL can't.
for db in $(docker exec postgres psql -U postgres -At -c "SELECT datname FROM pg_database WHERE NOT datistemplate AND datname <> 'postgres'"); do
  docker exec postgres pg_dump -U postgres -Fc "$db" > "$TMP/$db-$STAMP.dump"
done

docker run --rm --user root -v /etc/rclone:/config/rclone -v "$TMP":/backup rclone/rclone copy /backup garage-pg:postgres-backups -v
docker run --rm --user root -v /etc/rclone:/config/rclone rclone/rclone delete garage-pg:postgres-backups --min-age 30d
```

`/etc/systemd/system/postgres-backup.service`:

```ini
[Unit]
Description=Dump Postgres databases to Garage S3
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
ExecStart=/volume1/docker/postgres/backup.sh
```

`/etc/systemd/system/postgres-backup.timer` — 04:30, an hour before the HA sync so the two
backup jobs don't contend for NAS I/O:

```ini
[Unit]
Description=Nightly Postgres backup to Garage

[Timer]
OnCalendar=*-*-* 04:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now postgres-backup.timer
sudo systemctl list-timers postgres-backup.timer
```

!!! warning "systemd unit files: no inline comments, one unit per file"
    systemd only honors comments on their own line — an inline `# …` after a value becomes
    *part of the value*. And the `.service` and `.timer` are two separate files; pasting
    both into one produces `Unknown section 'Timer'` and a unit that can't start. Same
    UGOS caveat as the HA sync: these `/etc` units live outside Ansible's reach and a major
    firmware update can reset them — this page is the recovery reference.

Like every rclone job in this stack: `copy` + age-based `delete`, **never `sync`** — sync
mirrors deletions, so an empty source would wipe the Garage copy too.

## The restore drill — before the first tenant

A backup pipeline that has never restored anything is a hypothesis, not a backup. Prove it
with seeded data **before any app depends on this server**, and gate on restored *content*
— never on exit codes or "Completed" statuses (the same lesson the velero drill in the
[storage architecture](storage-architecture.md#the-local-path-tier) taught).

```bash
# Seed
sudo docker exec -ti postgres psql -U postgres -c 'CREATE DATABASE drilltest'
sudo docker exec -ti postgres psql -U postgres -d drilltest \
  -c 'CREATE TABLE t (id int, v text); INSERT INTO t SELECT g, md5(g::text) FROM generate_series(1,5000) g'

# Run the real pipeline, then confirm the bucket has objects with real sizes
sudo systemctl start postgres-backup.service
sudo docker exec -ti garage /garage bucket info postgres-backups   # Objects ≥ 2

# Restore FROM THE GARAGE COPY into a scratch database
mkdir -p /tmp/pg-drill
sudo docker run --rm --user root -v /etc/rclone:/config/rclone -v /tmp/pg-drill:/restore rclone/rclone copy garage-pg:postgres-backups /restore -v
sudo docker exec -ti postgres psql -U postgres -c 'CREATE DATABASE drillrestore'
sudo docker exec -i postgres pg_restore -U postgres -d drillrestore < /tmp/pg-drill/drilltest-*.dump

# THE GATE: row count and content checksum must match the source exactly
sudo docker exec -ti postgres psql -U postgres -d drilltest -c "SELECT count(*), md5(string_agg(v, ',' ORDER BY id)) FROM t"
sudo docker exec -ti postgres psql -U postgres -d drillrestore -c "SELECT count(*), md5(string_agg(v, ',' ORDER BY id)) FROM t"

# Clean up so the server is empty before its first real tenant
sudo docker exec -ti postgres psql -U postgres -c 'DROP DATABASE drillrestore'
sudo docker exec -ti postgres psql -U postgres -c 'DROP DATABASE drilltest'
rm -rf /tmp/pg-drill
```

Repeat the restore half of this drill monthly against a real app database (restore into a
scratch database, sanity-check row counts, drop it).

!!! note "Dump filenames and the NAS clock"
    `date +%F` in the script uses NAS-local time, which may be a day ahead of UTC — don't
    be surprised when the filename's date doesn't match a UTC `docker logs` timestamp.

## What about MariaDB?

Deferred until an app actually forces it (BookStack is the only confirmed one). When it
happens: a second compose service on another port following this exact shape — same hba
discipline (`mysql` has no pg_hba, use `bind-address` + per-user `@'10.0.20.1%'` grants),
same per-database `mysqldump` → Garage timer. Don't build it speculatively.

Point-in-time recovery (pgBackRest / WAL archiving) is likewise deferred — nightly logical
dumps match the current RPO. Revisit if an app's data stops being reconstructible from a
day-old copy.

## Verification

- [ ] `sudo docker exec -ti postgres psql -U postgres -c 'select version();'` → PostgreSQL 18.x
- [ ] `show data_checksums` → `on`
- [ ] `sudo ss -tlnp | grep 5433` → listener on `10.0.20.50:5433` only
- [ ] Cluster gate: `kubectl run pg-test …` → `FATAL: no pg_hba.conf entry for host "10.0.20.1x"`
- [ ] `sudo systemctl list-timers postgres-backup.timer` → a NEXT run is scheduled
- [ ] `garage bucket info postgres-backups` → Objects ≥ 2 with non-trivial sizes
- [ ] Restore drill: count **and** md5 identical between source and restored database
